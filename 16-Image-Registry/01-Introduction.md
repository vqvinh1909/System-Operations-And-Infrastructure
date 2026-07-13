---
title: "01 - Introduction"
module: 16
tags: [docker, sysops-infra, module-16, registry]
---

# 01 — Giới thiệu Module: Image Registry

## Tình huống mở đầu

Hãy tưởng tượng bạn là sysadmin duy nhất của một công ty 40 người. Team dev vừa build xong image `payment-service:v2.3` chứa toàn bộ logic xử lý thanh toán — bao gồm cả cách gọi tới API ngân hàng, cấu hình kết nối database nội bộ được bake sẵn trong image (một thói quen không tốt nhưng rất phổ biến ở giai đoạn đầu). Câu hỏi đặt ra: **image này sẽ được lưu ở đâu để 5 server production khác nhau đều pull được?**

Có ba lựa chọn, và mỗi lựa chọn có hệ quả khác nhau:

1. Copy file tar image qua `scp` tới từng server — hoạt động, nhưng không scale, không có versioning, không ai biết server nào đang chạy version nào.
2. Push lên Docker Hub public — hoạt động, nhưng **bất kỳ ai trên internet cũng thấy được** cấu hình và logic nghiệp vụ nhạy cảm bên trong (trừ khi trả tiền cho private repo, và ngay cả vậy dữ liệu vẫn nằm ngoài hạ tầng công ty).
3. Dựng một registry riêng bên trong mạng nội bộ — đây là hướng đi mà module này sẽ dạy.

Module 16 dạy bạn xây dựng và vận hành lựa chọn thứ ba, đi từ mức đơn giản nhất (`registry:2`) tới mức doanh nghiệp thật sự dùng (Harbor).

## Vì sao module này quan trọng với vai trò Sysadmin/DevOps

Ở khóa Linux Sysadmin bạn đã học cách quản lý một server chạy ổn định. Ở Part I/II của khóa này bạn học Ansible (tự động hóa triển khai) và Docker (đóng gói ứng dụng). Nhưng có một câu hỏi chưa được trả lời: **image Docker mà Ansible/CI pipeline sẽ deploy đó, lấy từ đâu ra?** Đó chính là vai trò của registry — nó là "kho trung tâm" nằm giữa bước "build" và bước "deploy" trong bất kỳ pipeline CI/CD nào. Không có registry nội bộ, không có CI/CD thật sự đúng nghĩa doanh nghiệp.

Trong công việc thực tế, sysadmin/DevOps thường là người:
- Dựng và duy trì registry (uptime, dung lượng đĩa, backup).
- Cấu hình quyền truy cập (project nào ai được push/pull).
- Thiết lập chính sách quét lỗ hổng bảo mật (vulnerability scanning) trước khi image được phép deploy.
- Dọn dẹp image cũ (garbage collection) để registry không phình dung lượng vô hạn.

## Mục tiêu học tập

Sau module này, bạn sẽ:

1. Phân biệt rõ Docker Hub, Private Registry tự host, và Harbor — biết khi nào dùng cái nào.
2. Tự dựng được một private registry bằng image chính thức `registry:2`.
3. Cài đặt và cấu hình Harbor — registry doanh nghiệp phổ biến nhất hiện nay — bao gồm project, user, và replication cơ bản.
4. Hiểu khái niệm image scanning và vulnerability database, thực hành scan một image thật để tìm lỗ hổng.
5. Xử lý được các sự cố thường gặp: lỗi TLS khi push/pull, registry đầy dung lượng, image bị "dangling" không dọn được.

## Kiến thức nền cần có trước

- Đã hoàn thành Module 15 (Multi-container Applications) — hiểu `docker build`, `docker push`, `docker pull`, Docker Compose cơ bản.
- Hiểu khái niệm image layer, image tag từ Part II.
- Biết dùng `openssl` tạo certificate tự ký (đã học ở Linux Sysadmin, phần Security Hardening) — sẽ dùng lại khi cấu hình TLS cho registry.
