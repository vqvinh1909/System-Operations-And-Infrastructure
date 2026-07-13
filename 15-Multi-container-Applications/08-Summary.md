---
title: "08 - Summary"
module: 15
tags: [docker, sysops-infra, module-15, summary]
---

# 08 - Summary: Multi-container Applications — Tổng kết Part II

## Những gì đã học trong module này

- **5 thành phần, 5 vai trò rõ ràng**: Nginx (cổng vào, TLS, load balancing), Backend (nghiệp vụ, stateless), Database (dữ liệu bền vững, ACID), Redis (cache/session, đọc nhanh), RabbitMQ (message queue, xử lý bất đồng bộ).
- **Redis vs RabbitMQ**: khác bản chất — Redis phục vụ tốc độ đọc trong luồng đồng bộ, RabbitMQ tách tác vụ chậm ra khỏi luồng phản hồi chính, xử lý bởi worker độc lập.
- **Network phân vùng**: chỉ `backend` (ứng dụng) nằm ở cả `frontend` và `backend` network — nguyên tắc giảm bề mặt tấn công áp dụng cho toàn bộ stack thật, không chỉ ví dụ tối giản.
- **Healthcheck + `depends_on` là bắt buộc**, không phải tùy chọn, trong một stack nhiều dependency — thiếu nó gây race condition/crash loop gần như chắc chắn ở lần khởi động đầu.
- **Backup database đúng cách**: dùng công cụ dump logic (`pg_dump`/`mysqldump`) thay vì nén thô thư mục dữ liệu khi database đang chạy live.
- **Vận hành thực tế**: rolling update từng service độc lập, scale worker theo tải queue, xử lý sự cố khi một dependency tạm thời không khả dụng.

## Checklist tự đánh giá — cũng là checklist tổng kết Part II

- [ ] Vẽ và giải thích được kiến trúc VM vs Container từ Module 08.
- [ ] Giải thích chuỗi `dockerd → containerd → shim → runc` từ Module 09.
- [ ] Viết Dockerfile multi-stage tối ưu cache, bảo mật từ Module 10.
- [ ] Vận hành container: resource limit, restart policy, log rotation từ Module 11.
- [ ] Giải thích network namespace, bridge, DNS nội bộ từ Module 12.
- [ ] Backup/restore volume đúng quy trình từ Module 13.
- [ ] Viết compose.yaml production-grade từ Module 14.
- [ ] **Tự tay dựng được một stack 5-6 thành phần hoàn chỉnh, vận hành và troubleshoot được — Module 15.**

Nếu tick đủ 8 mục trên, bạn đã hoàn thành toàn bộ **Part II - Container Fundamentals** với đủ chiều sâu để tự tin bước sang các chủ đề nâng cao hơn.

## Liên kết tới Module tiếp theo

Trong suốt Part II, bạn đã dùng image từ Docker Hub (`nginx`, `postgres`, `redis`, `rabbitmq`) và tự build image riêng cho backend. [[../16-Image-Registry/README|Module 16 - Image Registry]] giải quyết câu hỏi ở quy mô tổ chức: làm sao quản lý vòng đời hàng trăm image nội bộ một cách có kiểm soát — registry riêng, chính sách versioning, quét lỗ hổng bảo mật (image scanning), chính sách dọn dẹp image cũ. Đây là bước chuyển từ "biết build và chạy container" sang "vận hành container ở quy mô doanh nghiệp".

> [!tip] Ghi nhớ cốt lõi của toàn Part II
> Container không phải phép màu — mọi thứ bạn vừa dựng (namespace cô lập, cgroup giới hạn, OverlayFS ghép layer, bridge network với DNS nội bộ, volume bền vững, compose.yaml khai báo trạng thái mong muốn) đều là các cơ chế cụ thể, có thể giải thích, có thể kiểm chứng bằng tay. Một Sysadmin/DevOps Engineer giỏi không phải người nhớ nhiều lệnh `docker`, mà là người hiểu đúng từng lớp bên dưới để chẩn đoán chính xác khi hệ thống không hoạt động như kỳ vọng.
