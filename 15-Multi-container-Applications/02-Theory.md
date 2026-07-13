---
title: "02 - Theory"
module: 15
tags: [docker, sysops-infra, module-15, theory, nginx, mysql, postgres, redis, rabbitmq]
---

# 02 - Theory: Vai trò và Cách phối hợp của 5 thành phần

## 1. Nginx Reverse Proxy — cổng vào duy nhất của hệ thống

### 1.1 Reverse proxy khác gì forward proxy

Forward proxy đứng giữa client và Internet, che giấu client. **Reverse proxy** đứng giữa client và một nhóm server backend, che giấu backend — client chỉ biết địa chỉ của reverse proxy, không biết (và không cần biết) có bao nhiêu backend, backend nằm ở đâu.

### 1.2 Vai trò cụ thể trong stack

- **Cổng vào duy nhất (single entry point)**: chỉ Nginx publish port ra ngoài (`-p 80:80`, `-p 443:443`), mọi thành phần khác (backend, database, Redis, RabbitMQ) hoàn toàn không lộ ra ngoài mạng công cộng — giảm bề mặt tấn công tối đa, đúng nguyên tắc network phân vùng đã học ở Module 12/14.
- **TLS termination**: xử lý mã hóa/giải mã HTTPS tại đây, backend chỉ cần giao tiếp HTTP thuần nội bộ (đơn giản hóa cấu hình từng backend, tập trung quản lý chứng chỉ TLS ở một chỗ duy nhất).
- **Load balancing**: nếu có nhiều instance backend (ví dụ scale từ 1 lên 3 container backend), Nginx phân phối request giữa chúng theo thuật toán cấu hình (round-robin mặc định).
- **Static file serving**: có thể phục vụ trực tiếp file tĩnh (ảnh, CSS, JS) mà không cần đi qua backend, giảm tải cho tầng ứng dụng.

### 1.3 Internal Working: `upstream` block

```nginx
upstream backend_pool {
    server backend:3000;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`backend` trong `server backend:3000` chính là tên service Compose — nhờ embedded DNS đã học ở Module 12/14, Nginx resolve tên này ra đúng IP nội bộ của container backend mà không cần biết IP cứng. `proxy_set_header X-Real-IP` quan trọng về mặt vận hành: nếu không set, log của backend sẽ chỉ thấy IP nội bộ của Nginx làm "client IP" cho mọi request, mất hoàn toàn thông tin IP khách hàng thật — một lỗi cấu hình rất phổ biến gây khó khăn khi điều tra sự cố/bảo mật sau này.

## 2. Web/App Backend — tầng nghiệp vụ

Backend là thành phần duy nhất trong stack **do chính team phát triển viết** (build từ Dockerfile riêng, không dùng image có sẵn) — mọi thành phần còn lại đều là image chính thức từ registry công khai. Backend đóng vai trò điều phối: nhận request từ Nginx, đọc/ghi database, đọc/ghi cache Redis, đẩy message vào RabbitMQ khi cần xử lý bất đồng bộ.

**Nguyên tắc thiết kế quan trọng**: backend **không** nên tự lưu trạng thái (state) trong bộ nhớ hay trong writable layer của chính nó — mọi state cần thiết phải nằm ở database (bền vững) hoặc Redis (tạm thời nhưng chia sẻ được). Đây là điều kiện bắt buộc để có thể scale backend thành nhiều instance mà không có instance nào "biết nhiều hơn" instance khác — đúng nguyên tắc stateless service của kiến trúc cloud-native.

## 3. Database — MySQL/Postgres, tầng dữ liệu bền vững

### 3.1 Vì sao cần named volume tuyệt đối bắt buộc

Đã học kỹ ở Module 13: writable layer container không phải nơi lưu dữ liệu lâu dài. Với database, đây không phải một lựa chọn tùy chọn mà là **yêu cầu bắt buộc không có ngoại lệ** — mất dữ liệu database gần như luôn đồng nghĩa với mất dữ liệu khách hàng, giao dịch, không thể phục hồi.

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - db_data:/var/lib/postgresql/data
```

### 3.2 Healthcheck cho database — không thể bỏ qua trong stack nhiều thành phần

Đã học ở Module 14: `depends_on` short syntax chỉ đảm bảo thứ tự start, không đảm bảo database thực sự sẵn sàng nhận kết nối. Trong một stack 5 thành phần, backend phụ thuộc trực tiếp vào database — thiếu `healthcheck` + `condition: service_healthy` gần như chắc chắn gây crash loop ở lần khởi động đầu tiên của toàn bộ stack.

## 4. Redis — Cache và Session Store

### 4.1 Cơ chế

Redis lưu dữ liệu dạng key-value hoàn toàn (hoặc chủ yếu) trong RAM — tốc độ đọc/ghi tính bằng micro-giây, nhanh hơn database dựa trên đĩa hàng chục đến hàng trăm lần cho cùng một truy vấn.

### 4.2 Hai use case chính trong stack

- **Cache**: lưu kết quả của các truy vấn database tốn kém nhưng ít thay đổi (ví dụ danh sách sản phẩm nổi bật, thông tin cấu hình hệ thống). Backend kiểm tra Redis trước — nếu có (cache hit), trả về ngay, không chạm database; nếu không (cache miss), query database rồi lưu lại vào Redis cho lần sau (kèm thời gian hết hạn — TTL — để dữ liệu cache không bị "cũ" vĩnh viễn).
- **Session store**: lưu trạng thái đăng nhập/giỏ hàng tạm thời của người dùng, chia sẻ được giữa nhiều instance backend (khác với lưu session trong bộ nhớ của chính backend — cách này khiến người dùng bị "đăng xuất" ngẫu nhiên nếu request lần sau rơi vào một instance backend khác không có session đó trong bộ nhớ).

### 4.3 Persistence tùy chọn

Redis mặc định lưu dữ liệu trong RAM, có thể mất khi container restart nếu không cấu hình persistence (RDB snapshot hoặc AOF log ghi ra volume). Với vai trò cache thuần túy, mất dữ liệu khi restart **chấp nhận được** (dữ liệu sẽ được tính lại từ database ở lần truy vấn kế tiếp) — đây là lý do nhiều stack không cần mount volume cho Redis khi chỉ dùng làm cache. Nếu dùng Redis cho dữ liệu quan trọng hơn (ví dụ session không muốn mất khi restart), cần bật persistence và mount volume tương tự database.

## 5. RabbitMQ — Message Queue, xử lý bất đồng bộ

### 5.1 Mô hình Producer - Queue - Consumer

```
Backend (Producer) --publish--> [ RabbitMQ Queue ] --consume--> Worker (Consumer)
```

Backend (producer) đẩy một message vào queue (ví dụ "gửi email xác nhận đơn hàng #12345") rồi **trả response cho client ngay lập tức**, không chờ email thực sự được gửi xong. Một tiến trình worker riêng biệt (có thể là container riêng, hoặc cùng backend image chạy ở mode khác) liên tục lắng nghe queue, lấy message ra xử lý độc lập.

### 5.2 Lợi ích cụ thể

- **Giảm thời gian phản hồi cho người dùng**: tác vụ chậm (gửi email, gọi API bên thứ ba, xử lý ảnh) không làm chậm response chính.
- **Chịu lỗi tốt hơn**: nếu worker tạm thời lỗi/quá tải, message vẫn nằm an toàn trong queue, được xử lý lại khi worker phục hồi — không mất tác vụ như khi xử lý đồng bộ trực tiếp trong request (nếu request đó fail giữa chừng, tác vụ mất luôn).
- **Co giãn độc lập**: có thể tăng số lượng worker khi queue tồn đọng nhiều mà không cần đụng tới backend hay Nginx.

### 5.3 Management UI

RabbitMQ có image chính thức kèm sẵn giao diện quản trị web (`rabbitmq:4-management`), lắng nghe port 15672 mặc định — cho phép xem trực quan số lượng message trong từng queue, tốc độ xử lý, kết nối đang có. Đây là công cụ chẩn đoán đầu tiên khi nghi ngờ hệ thống xử lý bất đồng bộ có vấn đề (queue tồn đọng, worker không tiêu thụ kịp).

## 6. Cách 5 thành phần phối hợp — bức tranh tổng thể

```
Client -> Nginx (reverse proxy, TLS, load balancing)
            -> Backend (nghiệp vụ, stateless)
                -> Database (dữ liệu bền vững, ACID)
                -> Redis (cache/session, đọc nhanh)
                -> RabbitMQ (đẩy tác vụ bất đồng bộ)
                      -> Worker (tiêu thụ message, xử lý nền)
```

Mỗi mũi tên là một quyết định kiến trúc có lý do rõ ràng — không phải "thêm cho đủ công nghệ trendy". Xem sơ đồ trực quan đầy đủ (đã đo bằng script Python) ở [[03-Architecture]].
