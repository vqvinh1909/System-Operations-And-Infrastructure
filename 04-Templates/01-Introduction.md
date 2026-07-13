---
title: "01 - Introduction"
module: 4
tags: [ansible, sysops-infra, module-04, introduction]
---

# 01 — Giới thiệu

## Bài toán: 20 web server, 20 file cấu hình gần giống nhau

Bạn có 20 web server, mỗi server cần một file `nginx.conf` gần như giống hệt nhau — chỉ khác `server_name` (tên miền riêng) và `listen` port (một số server chạy port 80, số khác chạy 8080 do dùng chung máy với service khác). Với `copy` (Module 03), bạn buộc phải tạo **20 file tĩnh riêng biệt**, giống nhau 95% nội dung — và khi cần đổi một dòng chung cho cả 20 server (ví dụ tăng `worker_connections`), bạn phải sửa tay cả 20 file. Đây là kiểu công việc dễ sai sót, khó bảo trì, và đi ngược hoàn toàn tinh thần "single source of truth" đã học ở Module 00.

## Giải pháp: một Template, N kết quả

Với Jinja2 Template, bạn viết **một file `.j2` duy nhất**:

```jinja2
server {
    listen {{ http_port }};
    server_name {{ domain_name }};
    root /var/www/{{ domain_name }};
}
```

Ansible render file này riêng cho từng host, thay `{{ http_port }}` và `{{ domain_name }}` bằng giá trị biến thật của host đó (lấy từ inventory, `group_vars`, `host_vars` — đã học ở Module 02). Sửa một dòng logic chung, áp dụng ngay cho toàn bộ 20 server ở lần chạy Playbook tiếp theo.

## Module này khác Module 01 (Jinja2 Basics) ở đâu

Module 01 chỉ giới thiệu cú pháp `{{ }}`, `{% %}`, `{# #}` ở mức nhận diện — đủ để không bỡ ngỡ khi đọc thấy trong Playbook. Module này đi sâu: viết file `.j2` hoàn chỉnh với vòng lặp/điều kiện phức tạp, dùng Filter biến đổi dữ liệu, dùng Lookup lấy dữ liệu ngoài Ansible, và làm chủ trọn bộ 4 module xử lý file (`file`, `copy`, `fetch`, `synchronize`) — kỹ năng bắt buộc để triển khai cấu hình thật trong doanh nghiệp.

## Mục tiêu học tập

Sau module này, bạn viết được template `.j2` cho một file cấu hình dịch vụ thật (web server, có thể mở rộng cho các dịch vụ khác), dùng Filter và Lookup thành thạo, và biết chọn đúng module trong 4 module xử lý file cho từng tình huống cụ thể — chuẩn bị cho Module 05 nơi bạn sẽ học cách bảo vệ các giá trị nhạy cảm (mật khẩu, API key) dùng trong chính những template này.
