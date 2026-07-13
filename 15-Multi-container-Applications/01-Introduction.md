---
title: "01 - Introduction"
module: 15
tags: [docker, sysops-infra, module-15, introduction]
---

# 01 - Introduction: Vì sao cần 5 thành phần này cùng lúc

## Bài toán doanh nghiệp: một sàn thương mại điện tử điển hình

Hãy hình dung bạn được giao vận hành hạ tầng cho một sàn thương mại điện tử cỡ vừa. Yêu cầu nghiệp vụ:

- Khách hàng truy cập website qua HTTPS, cần chứng chỉ TLS quản lý tập trung, cần cân bằng tải khi có nhiều instance backend.
- Ứng dụng backend xử lý nghiệp vụ: giỏ hàng, đơn hàng, thanh toán — cần lưu dữ liệu **bền vững, có tính toàn vẹn giao dịch** (ACID).
- Trang sản phẩm được xem hàng triệu lượt/ngày — query thẳng vào database mỗi lần sẽ làm database quá tải, cần một lớp cache đọc nhanh.
- Khi khách đặt hàng, cần gửi email xác nhận, cập nhật kho, tính điểm thưởng — những việc này **không cần xảy ra ngay lập tức** trong lúc khách đang chờ trang phản hồi, nên cần xử lý bất đồng bộ ở nền.

Bốn yêu cầu trên ánh xạ trực tiếp tới 5 thành phần kỹ thuật:

| Yêu cầu nghiệp vụ | Thành phần kỹ thuật |
|---|---|
| Nhận traffic HTTPS, cân bằng tải, che giấu backend thật | **Nginx Reverse Proxy** |
| Xử lý nghiệp vụ chính | **Web/App Backend** |
| Lưu dữ liệu bền vững, giao dịch toàn vẹn | **Database (MySQL/Postgres)** |
| Đọc nhanh dữ liệu hay truy cập, giảm tải database | **Redis** |
| Xử lý tác vụ bất đồng bộ, tách rời khỏi luồng phản hồi chính | **RabbitMQ** |

Đây không phải kiến trúc lý thuyết — đây là bộ khung xuất hiện trong phần lớn hệ thống web application production thực tế, từ startup nhỏ tới tập đoàn lớn, chỉ khác nhau ở quy mô và công cụ orchestration (Compose cho 1 host, Kubernetes cho nhiều host — sẽ học ở khóa sau).

## Vì sao không gộp mọi thứ vào một container duy nhất

Người mới thường đặt câu hỏi: "sao không viết một Dockerfile cài hết Nginx + app + MySQL + Redis + RabbitMQ vào một container cho gọn?" Đây là **anti-pattern** nghiêm trọng, vì:

- **Vi phạm nguyên tắc một tiến trình chính mỗi container** (đã học Module 09-11): PID 1 phức tạp, signal handling rối, healthcheck không thể phản ánh đúng trạng thái riêng từng thành phần.
- **Không scale độc lập được**: traffic tăng cần thêm backend, không cần thêm Nginx hay database — tách container cho phép scale đúng đúng thành phần cần thiết.
- **Update một thành phần buộc rebuild/restart toàn bộ**: sửa version Redis buộc restart cả Nginx lẫn database, gây downtime không cần thiết.
- **Không tận dụng được image chính thức đã tối ưu sẵn** (đã học Module 10): `postgres:16-alpine`, `redis:7-alpine`, `rabbitmq:4-management` đều là image chính thức, bảo trì tốt, đã qua kiểm định bảo mật — tự build lại từ đầu là lãng phí và rủi ro hơn.

## Vì sao Redis và RabbitMQ không thay thế được nhau

Đây là điểm hay bị nhầm lẫn nhất khi mới tiếp cận kiến trúc này:

- **Redis**: là kho lưu trữ key-value trong bộ nhớ (in-memory), cực nhanh, dùng cho **cache** (giảm tải query lặp lại tới database) và **session store** (lưu trạng thái đăng nhập, giỏ hàng tạm). Redis trả lời câu hỏi "làm sao đọc dữ liệu nhanh hơn".
- **RabbitMQ**: là message broker, triển khai mô hình **hàng đợi tin nhắn** (message queue) — nơi một thành phần "gửi" một tác vụ (message) vào hàng đợi, một hoặc nhiều "worker" khác tiêu thụ (consume) và xử lý tác vụ đó độc lập, không đồng bộ với luồng xử lý request ban đầu. RabbitMQ trả lời câu hỏi "làm sao tách một tác vụ chậm ra khỏi luồng phản hồi nhanh cho người dùng".

Redis **có thể** dùng tạm làm hàng đợi đơn giản (qua cấu trúc List/Stream), và RabbitMQ **có thể** dùng tạm làm cache (không hợp lý về mặt thiết kế) — nhưng mỗi công cụ được tối ưu cho đúng một bài toán, dùng sai mục đích sẽ gặp giới hạn khi hệ thống scale lớn hơn.

## Mục tiêu học tập của module

Sau module này, bạn có thể:

1. Vẽ và giải thích kiến trúc 5 thành phần, chỉ đúng vai trò và luồng giao tiếp giữa chúng.
2. Viết `compose.yaml` hoàn chỉnh dựng toàn bộ stack, tuân thủ mọi best practice đã học ở Module 08-14.
3. Thiết kế network phân vùng đúng nguyên tắc bảo mật (chỉ Nginx được publish port ra ngoài).
4. Vận hành, giám sát, và khắc phục sự cố cho một hệ thống nhiều thành phần phối hợp — kỹ năng khác biệt với việc vận hành một container đơn lẻ.

## Liên kết

Module này tổng hợp toàn bộ Part II. Đọc tiếp [[02-Theory]] để đi vào chi tiết vai trò kỹ thuật và cách phối hợp của từng thành phần.
