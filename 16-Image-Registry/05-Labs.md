---
title: "05 - Labs"
module: 16
tags: [docker, sysops-infra, module-16, registry, harbor, lab]
---

# 05 — Bài Lab Thực Hành

> Lưu toàn bộ file cấu hình/kết quả vào [[labs/README|labs/]] theo đúng cấu trúc gợi ý.

## Lab 1 (Beginner): Dựng Private Registry bằng `registry:2`

**Mục tiêu**: Có một registry nội bộ hoạt động được, push/pull thành công một image thật.

1. Chạy container `registry:2` theo lệnh ở [[04-Commands]] (không TLS, chỉ để làm quen — ghi rõ trong `notes.md` rằng đây là cấu hình KHÔNG an toàn cho production).
2. Pull image `alpine:3.20` từ Docker Hub, gắn tag trỏ về `localhost:5000`, push lên registry vừa dựng.
3. Xóa image `alpine:3.20` gắn tag registry khỏi local (`docker rmi`), sau đó pull lại từ `localhost:5000` để chứng minh registry hoạt động độc lập với Docker Hub.
4. Dùng `curl` gọi API `_catalog` và `tags/list` để liệt kê nội dung registry mà không cần Docker CLI — chứng minh registry là một HTTP API chuẩn.

**Tiêu chí hoàn thành**: Pull thành công từ registry nội bộ sau khi đã xóa image gốc; giải thích được (bằng lời của bạn, viết vào `notes.md`) vì sao bước xóa rồi pull lại là bước kiểm chứng quan trọng.

## Lab 2 (Intermediate): Bảo mật Private Registry bằng TLS + Basic Auth

**Mục tiêu**: Chuyển registry ở Lab 1 từ "mở hoàn toàn" sang có tối thiểu TLS và xác thực.

1. Tạo self-signed certificate bằng `openssl` (đã học ở Linux Sysadmin — Security Hardening).
2. Chạy lại registry với `REGISTRY_HTTP_TLS_CERTIFICATE`/`REGISTRY_HTTP_TLS_KEY`.
3. Thêm Basic Auth bằng `htpasswd`, tạo ít nhất 2 user khác nhau.
4. Thử `docker login` bằng user sai mật khẩu — quan sát lỗi trả về, ghi lại nguyên văn message lỗi vào `notes.md`.
5. Thử push khi **chưa** login — quan sát và ghi lại lỗi.

**Tiêu chí hoàn thành**: Push/pull chỉ thành công sau khi login đúng; push bị từ chối khi chưa login hoặc sai mật khẩu; certificate tự ký được client tin cậy (hoặc bạn giải thích đúng vì sao phải dùng `insecure-registries` nếu bỏ qua bước trust certificate).

## Lab 3 (Advanced): Cài đặt Harbor và tổ chức Project/RBAC

**Mục tiêu**: Có một Harbor instance chạy được, mô phỏng đúng mô hình một doanh nghiệp nhỏ có 2 team.

1. Cài Harbor bằng offline installer, bật `--with-trivy`.
2. Tạo 2 project: `team-backend` (private) và `team-frontend` (private).
3. Tạo 2 user, gán mỗi user vào một project với vai trò `Developer`.
4. Đăng nhập bằng user của `team-backend`, thử push vào `team-frontend` — quan sát bị từ chối, ghi lại đúng thông báo lỗi.
5. Tạo một **robot account** cho `team-backend`, dùng robot account đó (không dùng tài khoản cá nhân) để push một image qua script/CLI — đây là cách làm đúng chuẩn CI/CD.

**Tiêu chí hoàn thành**: RBAC hoạt động đúng (không cross-project được); robot account push thành công độc lập với tài khoản người dùng thật.

## Lab 4 (Advanced): Image Scanning và Policy chặn deploy

**Mục tiêu**: Thực hành quy trình bảo mật image thật, không chỉ lý thuyết.

1. Build một image cố ý dùng base image cũ, nhiều lỗ hổng đã biết (ví dụ `FROM ubuntu:18.04` hoặc một phiên bản `node` cũ) để có kết quả scan phong phú.
2. Push image này lên Harbor, để tính năng auto-scan chạy (hoặc kích hoạt scan thủ công qua UI/API).
3. Xem báo cáo scan trên Portal — liệt kê ít nhất 3 CVE, phân loại theo mức độ (Critical/High/Medium/Low), ghi lại vào `notes.md`.
4. Bật policy "Prevent vulnerable images from running" ở mức Critical trong cấu hình project, thử pull lại image đó, quan sát bị chặn.
5. So sánh kết quả scan trên với kết quả `docker scout cves` chạy trên cùng image — hai công cụ có ra danh sách giống hệt nhau không? Ghi nhận xét vào `notes.md` (gợi ý: khác nhau vì mỗi công cụ dùng tập vulnerability database và logic phân tích khác nhau, đây là lý do nhiều doanh nghiệp dùng song song hơn một scanner).

**Tiêu chí hoàn thành**: Có báo cáo scan thật với ít nhất 3 CVE cụ thể; chứng minh được policy chặn hoạt động (pull bị từ chối); có so sánh Trivy vs Docker Scout bằng dữ liệu thật, không suy đoán.

## Lab 5 (Enterprise, tùy chọn): Replication giữa 2 Harbor instance

**Mục tiêu**: Mô phỏng kịch bản doanh nghiệp có 2 site (ví dụ site chính và site DR).

1. Dựng thêm một Harbor instance thứ hai (có thể trên VM/container khác, đổi cổng để tránh xung đột).
2. Cấu hình một "Registry Endpoint" trỏ từ Harbor A sang Harbor B (Administration → Registries).
3. Tạo Replication Rule: mọi image push vào project `team-backend` ở Harbor A tự động đồng bộ sang Harbor B.
4. Push một image mới, xác nhận nó xuất hiện ở Harbor B sau khi replication job chạy xong.

**Tiêu chí hoàn thành**: Image xuất hiện đúng ở Harbor B mà không cần push thủ công lần hai; giải thích được (viết vào `notes.md`) đây là cơ chế nào trong kiến trúc Harbor đã học ở [[03-Architecture]] (gợi ý: Job Service).
