---
title: "06 - Troubleshooting"
module: 16
tags: [docker, sysops-infra, module-16, registry, harbor, troubleshooting]
---

# 06 — Lỗi Thường Gặp và Cách Sửa

## Lỗi 1: `http: server gave HTTP response to HTTPS client`

**Triệu chứng**: `docker push registry.company.local:5000/app:v1` báo lỗi này ngay lập tức.

**Nguyên nhân**: Docker daemon mặc định yêu cầu HTTPS cho mọi registry không phải `localhost`. Registry của bạn đang chạy HTTP thuần (chưa cấu hình TLS).

**Cách sửa**:
- Cách đúng lâu dài: cấu hình TLS thật cho registry (xem [[04-Commands]]).
- Cách tạm cho lab: thêm registry vào danh sách `insecure-registries` trong `/etc/docker/daemon.json` của **mọi client** cần push/pull (không chỉ server chạy registry), rồi `sudo systemctl restart docker`.

## Lỗi 2: `x509: certificate signed by unknown authority`

**Triệu chứng**: Push/pull báo lỗi certificate ngay cả khi registry đã bật TLS.

**Nguyên nhân**: Bạn dùng self-signed certificate — client không có CA nào để xác minh certificate này là hợp lệ (khác với certificate từ Let's Encrypt hay CA nội bộ doanh nghiệp mà máy đã trust sẵn).

**Cách sửa**: Copy file `domain.crt` vào đúng thư mục trust certificate của Docker trên client:

```bash
sudo mkdir -p /etc/docker/certs.d/registry.company.local:443
sudo cp domain.crt /etc/docker/certs.d/registry.company.local:443/ca.crt
sudo systemctl restart docker
```

Lưu ý tên thư mục phải khớp chính xác `host:port` bạn dùng để push/pull, sai một ký tự (thiếu port, sai hostname) là Docker sẽ không áp dụng certificate này.

## Lỗi 3: `unauthorized: authentication required`

**Triệu chứng**: Push bị từ chối dù bạn chắc chắn đã `docker login` thành công trước đó.

**Nguyên nhân thường gặp nhất**: Token đăng nhập hết hạn (session timeout), hoặc bạn login vào một registry nhưng đang thao tác với registry/project khác (nhầm domain/project trong tag). Cũng có thể do user có quyền `Guest` (chỉ pull) chứ không phải `Developer` (được push).

**Cách chẩn đoán**:
```bash
docker login registry.company.local   # login lai, xem co bao loi khong
docker inspect --format='{{.RepoTags}}' <image-id>   # kiem tra dung tag chua, dung project chua
```
Trên Harbor: vào Portal → Project → Members, kiểm tra vai trò của user đang dùng — đây là nguyên nhân phổ biến hơn nhiều so với lỗi xác thực thật sự.

## Lỗi 4: Registry đầy dung lượng đĩa, push bắt đầu fail

**Triệu chứng**: `docker push` báo lỗi `no space left on device`, hoặc Harbor UI cảnh báo disk usage cao.

**Nguyên nhân**: Registry/Harbor tích lũy blob theo thời gian — mỗi lần push một image mới (kể cả chỉ đổi 1 dòng code, rebuild toàn bộ), các layer cũ **không tự động bị xóa** trừ khi bạn chủ động xóa tag cũ và chạy Garbage Collection.

**Cách xử lý**:
1. Xác định dung lượng đang dùng: `df -h` trên host, hoặc Harbor UI → Projects → xem dung lượng từng project.
2. Xóa các tag không còn cần (image test cũ, image của nhánh feature đã merge xong) qua UI hoặc API.
3. Chạy Garbage Collection (xem [[04-Commands]]) — **đây là bước bắt buộc**, xóa tag thôi chưa giải phóng dung lượng vì blob vẫn còn tồn tại tới khi GC quét và xóa blob không còn manifest nào tham chiếu.
4. Về lâu dài: thiết lập **tag retention policy** trên Harbor (tự động xóa image quá cũ theo số lượng tag giữ lại hoặc theo tuổi) để không phải dọn tay định kỳ.

> [!warning] Xóa tag không giải phóng dung lượng ngay
> Đây là hiểu lầm phổ biến nhất của người mới quản trị registry: xóa tag (hoặc xóa repository) trên Harbor UI chỉ đánh dấu manifest là "unreferenced", **không xóa blob thật trên đĩa**. Dung lượng chỉ thực sự được giải phóng sau khi Garbage Collection chạy xong.

## Lỗi 5: `docker push` bị treo (hang) giữa chừng khi upload layer lớn

**Triệu chứng**: Push một image có layer vài GB, tiến trình dừng giữa chừng không báo lỗi rõ ràng, hoặc timeout sau vài phút.

**Nguyên nhân thường gặp**: Reverse proxy (Nginx) đặt trước registry có giới hạn `client_max_body_size` mặc định quá nhỏ (thường 1MB), hoặc timeout proxy quá ngắn so với thời gian upload thực tế trên đường truyền chậm.

**Cách sửa**: Nếu bạn tự đặt Nginx trước `registry:2` (mô hình phổ biến để thêm TLS/rate-limit mà không sửa registry), cần tăng các tham số sau trong cấu hình Nginx:
```nginx
client_max_body_size 0;        # khong gioi han kich thuoc body
proxy_read_timeout 900;
proxy_send_timeout 900;
```
Harbor cài qua offline installer đã tự cấu hình các tham số này hợp lý — lỗi này chủ yếu gặp khi bạn tự dựng `registry:2` phía sau reverse proxy tự viết.

## Lỗi 6: Harbor Core lên nhưng không kết nối được PostgreSQL/Redis

**Triệu chứng**: `docker compose ps` trong thư mục cài Harbor cho thấy container `harbor-core` liên tục restart.

**Cách chẩn đoán**: Đây là lúc kiến trúc đã học ở [[03-Architecture]] hữu ích thật sự — xem log của đúng container đang lỗi, không đoán mò:
```bash
docker compose logs harbor-core --tail=50
docker compose logs harbor-db --tail=50
docker compose ps    # kiem tra harbor-db, redis co dang "healthy" khong
```
Nguyên nhân thường gặp: cấu hình `harbor.yml` bị sửa sau khi đã chạy `install.sh` lần đầu mà không chạy lại đúng quy trình (`./prepare` trước khi `docker compose up` lại) — Harbor không tự áp dụng thay đổi `harbor.yml` giữa chừng như một service thông thường, vì `install.sh`/`prepare` sinh ra các file cấu hình runtime cho từng container từ `harbor.yml`.

## Lỗi 7: Scan job trong Harbor treo ở trạng thái "Pending" mãi không chạy

**Triệu chứng**: Push image lên, trạng thái scan hiển thị "Pending" không bao giờ chuyển sang "Success"/"Fail".

**Nguyên nhân thường gặp**: Chưa cấu hình scanner (nếu cài Harbor không dùng cờ `--with-trivy` và chưa đăng ký scanner thủ công), hoặc Job Service không kết nối được Redis (job queue nằm ở Redis, xem sơ đồ ở [[03-Architecture]]).

**Cách chẩn đoán**:
```bash
docker compose logs harbor-jobservice --tail=50
docker compose logs redis --tail=50
```
Kiểm tra Administration → Interrogation Services trên Portal — nếu không thấy scanner nào được đăng ký (Not Set), đây chính là nguyên nhân, cần đăng ký lại Trivy hoặc cài đặt lại với `--with-trivy`.

## Bảng tra nhanh: log của thành phần nào cần xem theo triệu chứng

| Triệu chứng | Container log cần xem trước |
|---|---|
| Không login được / push bị từ chối do quyền | `harbor-core` |
| Push OK nhưng blob không lưu được | `registry` (Distribution) |
| Portal load được nhưng thao tác project lỗi | `harbor-core`, `harbor-db` |
| Scan không bao giờ chạy | `harbor-jobservice`, `redis`, `trivy-adapter` |
| Toàn bộ Harbor không truy cập được | `nginx` (proxy trước Harbor), rồi mới tới các container còn lại |
