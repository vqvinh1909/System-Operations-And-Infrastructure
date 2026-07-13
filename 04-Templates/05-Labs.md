---
title: "05 - Labs"
module: 4
tags: [ansible, sysops-infra, module-04, labs]
---

# 05 — Labs thực hành

> Yêu cầu môi trường: control node + inventory Module 02-03 (2 web server khác `http_port`/`domain_name`).

## Lab 1 — Template nginx động theo host

**Mục tiêu:** một file `.j2` render ra cấu hình khác nhau cho từng web server.

**Các bước:**

1. Khai báo biến `http_port` và `domain_name` khác nhau cho `web-01` và `web-02` (trong inventory hoặc `host_vars/`).
2. Viết `templates/nginx.conf.j2` dùng `{{ http_port }}`, `{{ domain_name }}`.
3. Viết Task dùng module `template` render file này vào `/etc/nginx/sites-available/{{ domain_name }}.conf`, `notify` restart nginx.
4. Chạy Playbook, xác nhận 2 web server có file cấu hình nội dung khác nhau đúng theo biến đã khai báo.

**Kết quả cần có:** file `.j2`, 2 file cấu hình đã render trên 2 host khác nhau (copy về để đối chiếu).

## Lab 2 — Điều kiện và vòng lặp trong template

**Mục tiêu:** dùng `{% if %}`/`{% for %}` thật trong file `.j2`.

**Các bước:**

1. Thêm biến boolean `enable_ssl` (khác giá trị giữa 2 host), dùng `{% if enable_ssl %}` trong template để thêm khối cấu hình SSL chỉ khi biến này `true`.
2. Thêm biến list `upstream_servers` (danh sách IP backend), dùng `{% for %}` để render khối `upstream { ... }` liệt kê từng server.
3. Chạy `--check --diff` trước khi áp dụng thật, đọc kỹ phần diff sinh ra từ `{% if %}`/`{% for %}`.

**Kết quả cần có:** template cập nhật có cả `if` và `for`, output `--diff` đã lưu lại cho 2 trường hợp `enable_ssl: true` và `false`.

## Lab 3 — Filters thực chiến

**Mục tiêu:** dùng ít nhất 5 filter khác nhau trong một Playbook/template thật.

**Các bước:**

1. Dùng `default` cho một biến có thể chưa được khai báo ở một số host (ví dụ `max_connections | default(1024)`).
2. Dùng `join` để render danh sách package đã cài thành một dòng log dễ đọc.
3. Dùng `upper`/`lower` để chuẩn hóa một biến tên môi trường trước khi ghi vào file cấu hình.
4. Dùng `to_nice_yaml` để debug in ra toàn bộ một biến dictionary phức tạp qua module `debug`.
5. Dùng `map(attribute=...)` để trích danh sách tên từ một list dictionary user đã khai báo ở Module 03 Lab 2.

**Kết quả cần có:** log output của Task `debug` cho từng filter, giải thích ngắn kết quả mỗi filter làm gì với dữ liệu đầu vào.

## Lab 4 — Lookup lấy dữ liệu từ Control Node

**Mục tiêu:** hiểu Lookup chạy trên Control Node, khác với Facts/Task chạy trên Managed Node.

**Các bước:**

1. Dùng `lookup('file', ...)` đọc nội dung public key SSH automation (đã tạo ở Module 01), gán vào biến, in ra bằng `debug`.
2. Dùng `lookup('env', 'USER')` hoặc biến môi trường tương tự trên control node, in ra bằng `debug`.
3. Dùng `lookup('pipe', 'date +%F')` lấy ngày hiện tại từ control node, dùng giá trị này đặt tên file backup trong một Task `copy`/`template` (ví dụ `dest: /etc/nginx/backup-{{ lookup('pipe', 'date +%F') }}.conf`).

**Kết quả cần có:** log output 3 lookup, và 1 file backup được đặt tên có ngày tháng thật.

## Lab 5 — `file`, `fetch`, `synchronize` trong một quy trình triển khai

**Mục tiêu:** phối hợp cả 4 module xử lý file trong một Playbook hoàn chỉnh.

**Các bước:**

1. Dùng `file` tạo cấu trúc thư mục cho một ứng dụng giả lập (`/opt/myapp/releases`, `/opt/myapp/current` dạng symlink) với quyền phù hợp.
2. Dùng `synchronize` đồng bộ một thư mục nội dung mẫu (tạo sẵn trên control node) lên `/opt/myapp/releases/v1/`.
3. Dùng `file` tạo symlink `/opt/myapp/current` trỏ tới `/opt/myapp/releases/v1/`.
4. Dùng `fetch` lấy về control node một file log giả lập đã tạo trên managed node trong bước trên, xác nhận tên file lấy về có gắn tên host để không bị ghi đè giữa các host.

**Kết quả cần có:** cấu trúc thư mục trên managed node đúng như thiết kế, thư mục `collected-logs/` trên control node chứa file log riêng từng host.

## Checklist hoàn thành module

- [ ] Viết được template `.j2` render đúng biến khác nhau cho từng host.
- [ ] Dùng đúng `{% if %}`/`{% for %}` trong template, đóng tag đầy đủ.
- [ ] Dùng thành thạo ít nhất 5 filter khác nhau, giải thích được từng filter làm gì.
- [ ] Dùng Lookup lấy dữ liệu từ file/biến môi trường/lệnh trên Control Node.
- [ ] Phân biệt và dùng đúng cả 4 module: `file`, `copy`, `template`, `fetch`, `synchronize` trong một tình huống thực tế.
