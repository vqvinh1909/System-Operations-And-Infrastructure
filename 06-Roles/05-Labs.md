---
title: "05 - Labs"
module: 6
tags: [ansible, sysops-infra, module-06, labs]
---

# 05 — Labs thực hành

> Yêu cầu môi trường: control node + inventory/Playbook đã xây dựng qua các module trước.

## Lab 1 — Tạo Role từ đầu bằng `ansible-galaxy role init`

**Mục tiêu:** có một Role đúng cấu trúc chuẩn, sẵn sàng điền nội dung.

**Các bước:**

1. Chạy `ansible-galaxy role init nginx_hardened` trong thư mục `roles/` của dự án.
2. Liệt kê cấu trúc thư mục vừa tạo (`find nginx_hardened -type f`), đối chiếu với sơ đồ ở [[03-Architecture|03 - Architecture]].
3. Viết `README.md` trong Role mô tả ngắn gọn: Role này làm gì, biến nào có thể cấu hình.

**Kết quả cần có:** thư mục `roles/nginx_hardened/` đúng cấu trúc chuẩn, có `README.md`.

## Lab 2 — Chuyển Playbook cũ (Module 03-04) thành Role

**Mục tiêu:** thực hành refactor Task rời rạc thành Role tái sử dụng.

**Các bước:**

1. Lấy lại Task cài đặt + cấu hình nginx đã viết ở Module 03-04, di chuyển vào `roles/nginx_hardened/tasks/main.yml`.
2. Di chuyển file `.j2` vào `roles/nginx_hardened/templates/`, Handler vào `roles/nginx_hardened/handlers/main.yml`.
3. Xác định biến nào nên là `defaults` (người dùng có thể muốn đổi, ví dụ `http_port`) và biến nào nên là `vars` (cố định, ví dụ đường dẫn file cấu hình) — điền đúng vào 2 file tương ứng.
4. Viết `site.yml` mới, chỉ còn `roles: [nginx_hardened]`, chạy lại và xác nhận hành vi giống hệt Playbook cũ.

**Kết quả cần có:** `site.yml` rút gọn còn vài dòng, Role hoạt động đúng như Playbook gốc, log chạy xác nhận idempotent.

## Lab 3 — Role Dependencies

**Mục tiêu:** một Role tự động kéo theo Role khác chạy trước.

**Các bước:**

1. Tạo thêm một Role tối giản `common_hardening` (ví dụ chỉ có Task tắt password authentication SSH — mô phỏng, không cần đầy đủ).
2. Khai báo `common_hardening` là dependency của `nginx_hardened` trong `meta/main.yml`.
3. Chạy Playbook chỉ gọi `nginx_hardened` — quan sát log xác nhận `common_hardening` tự động chạy trước.
4. Thêm một Role thứ hai cũng phụ thuộc `common_hardening`, gọi cả hai Role trong cùng Playbook, xác nhận `common_hardening` chỉ chạy đúng một lần.

**Kết quả cần có:** `meta/main.yml` có khai báo `dependencies`, log chạy minh họa đúng thứ tự và việc không chạy lặp lại.

## Lab 4 — Cài và dùng Role/Collection từ Ansible Galaxy

**Mục tiêu:** tích hợp một Role bên thứ ba thật vào dự án.

**Các bước:**

1. Viết `requirements.yml` khai báo một Role phổ biến từ Galaxy (ví dụ Role quản lý user/package cơ bản), ghim version cụ thể.
2. Chạy `ansible-galaxy install -r requirements.yml`, xác nhận Role được tải về đúng thư mục.
3. Gọi Role đó trong một Playbook thử nghiệm riêng, chạy trên 1 host lab.
4. Đọc `README.md` của Role đó (nếu có) để hiểu biến cấu hình, thử ghi đè ít nhất 1 biến qua `vars:` khi gọi Role.

**Kết quả cần có:** `requirements.yml`, log cài đặt thành công, Playbook thử nghiệm chạy đúng với Role bên ngoài.

## Lab 5 — Test Role độc lập trước khi ráp vào hệ thống lớn

**Mục tiêu:** luyện thói quen kiểm thử cô lập.

**Các bước:**

1. Viết một Playbook tối giản `test-nginx-role.yml` chỉ gọi đúng `roles: [nginx_hardened]`, không có Task nào khác.
2. Chạy `--check --diff` rồi chạy thật trên một host lab riêng (không phải host đang chạy production giả lập của các lab trước).
3. Cố tình gây lỗi nhỏ trong Role (ví dụ đường dẫn sai), chạy lại `test-nginx-role.yml`, xác nhận lỗi được phát hiện nhanh và rõ ràng nhờ phạm vi test đã bị cô lập, không lẫn với Task khác.

**Kết quả cần có:** `test-nginx-role.yml`, log lỗi rõ ràng khi cố tình gây lỗi, log thành công sau khi sửa.

## Checklist hoàn thành module

- [ ] Tạo Role đúng cấu trúc chuẩn bằng `ansible-galaxy role init`.
- [ ] Chuyển đổi thành công một Playbook cũ thành Role, giữ nguyên hành vi.
- [ ] Phân biệt và dùng đúng `defaults` vs `vars` khi thiết kế Role.
- [ ] Khai báo và xác nhận hoạt động của Role Dependencies qua `meta/main.yml`.
- [ ] Cài và dùng thành công một Role/Collection từ Ansible Galaxy qua `requirements.yml`.
