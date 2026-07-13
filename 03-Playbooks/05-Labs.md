---
title: "05 - Labs"
module: 3
tags: [ansible, sysops-infra, module-03, labs]
---

# 05 — Labs thực hành

> Yêu cầu môi trường: control node + inventory từ Module 02 (2 web server, 1 db server).

## Lab 1 — Playbook đầu tiên: cài đặt web server cơ bản

**Mục tiêu:** viết một Playbook nhiều Task hoàn chỉnh, chạy được bằng `ansible-playbook`.

**Các bước:**

1. Viết `site.yml` với Play nhắm vào `webservers`: cài `nginx`, tạo thư mục `/var/www/lab03`, copy một file `index.html` tuỳ chọn vào đó, đảm bảo service `nginx` đang chạy và bật tự khởi động.
2. Chạy `ansible-playbook site.yml --check --diff` trước, đọc kỹ output dự kiến thay đổi.
3. Chạy thật `ansible-playbook site.yml`, xác nhận trên trình duyệt/`curl` truy cập được `index.html` qua IP của web server.
4. Chạy lại lần thứ hai, xác nhận toàn bộ Task báo `changed=0` (idempotent).

**Kết quả cần có:** file `site.yml`, log 2 lần chạy (lần đầu có `changed`, lần hai không).

## Lab 2 — Loop và Conditional

**Mục tiêu:** dùng `loop` để tránh lặp code, dùng `when` để playbook chạy đúng theo hệ điều hành.

**Các bước:**

1. Sửa Task cài package trong `site.yml` để cài đồng thời 3 package (`nginx`, `git`, `curl`) bằng `loop`, thay vì viết 3 Task riêng.
2. Thêm 2 Task cài package dùng module khác nhau theo `ansible_os_family` (`apt` cho Debian, `dnf`/`yum` cho RedHat), dùng `when` để chỉ 1 trong 2 chạy đúng trên mỗi host.
3. Thêm 1 Task tạo nhiều user bằng `loop` với list chứa dictionary (mỗi user có `name` và `groups` khác nhau).

**Kết quả cần có:** `site.yml` cập nhật, log chạy xác nhận đúng Task nào chạy trên host nào.

## Lab 3 — Handler cho việc restart service có điều kiện

**Mục tiêu:** thấy rõ Handler chỉ chạy khi thật sự có thay đổi.

**Các bước:**

1. Thêm Task copy file cấu hình nginx tuỳ chỉnh (`nginx.conf`), `notify` một Handler restart nginx.
2. Chạy Playbook lần đầu — quan sát Handler có chạy (vì file cấu hình vừa thay đổi).
3. Chạy lại lần hai không sửa gì — quan sát Handler **không chạy** (vì Task copy báo `changed=0`).
4. Sửa nhẹ nội dung file cấu hình, chạy lại lần ba — quan sát Handler chạy lại đúng một lần dù có thể có nhiều Task khác cũng notify cùng Handler này.

**Kết quả cần có:** 3 log chạy minh họa rõ hành vi Handler ở 3 tình huống khác nhau.

## Lab 4 — Block/Rescue/Always cho triển khai có xử lý lỗi

**Mục tiêu:** thực hành mô hình try/except/finally của Ansible.

**Các bước:**

1. Viết một Play riêng dùng `block` gồm: dừng service, cố tình chạy một Task sẽ thất bại (ví dụ lệnh không tồn tại, hoặc đường dẫn sai).
2. Thêm `rescue` in ra thông báo lỗi bằng `debug`, mô phỏng "rollback".
3. Thêm `always` khởi động lại service, chạy bất kể `block` thành công hay thất bại.
4. Chạy Playbook, xác nhận đúng thứ tự: block thất bại → rescue chạy → always vẫn chạy.
5. Sửa lại Task lỗi trong `block` thành hợp lệ, chạy lại, xác nhận `rescue` bị bỏ qua nhưng `always` vẫn chạy.

**Kết quả cần có:** 2 log chạy (một có lỗi kích hoạt rescue, một không lỗi) để đối chiếu.

## Lab 5 — Tags, Include/Import tổ chức Playbook lớn

**Mục tiêu:** tách một Playbook dài thành nhiều file, chạy chọn lọc bằng tags.

**Các bước:**

1. Tách `site.yml` hiện tại thành 2 file riêng: `tasks/install.yml` (các Task cài đặt package, đánh `tags: [install]`) và `tasks/configure.yml` (các Task copy cấu hình, đánh `tags: [config]`).
2. Trong `site.yml` chính, dùng `import_tasks` để gộp 2 file trên vào Play.
3. Chạy `ansible-playbook site.yml --tags config` — xác nhận chỉ Task cấu hình chạy, Task cài đặt bị bỏ qua.
4. Đổi 1 trong 2 `import_tasks` thành `include_tasks` với tên file có chứa biến (ví dụ `"{{ action_type }}.yml"`), truyền `action_type` qua `--extra-vars`, xác nhận vẫn hoạt động đúng.

**Kết quả cần có:** cấu trúc thư mục có `site.yml` + `tasks/install.yml` + `tasks/configure.yml`, log chạy với `--tags` xác nhận đúng hành vi chọn lọc.

## Checklist hoàn thành module

- [ ] Viết được Playbook nhiều Task, chạy idempotent (`changed=0` ở lần chạy lặp lại).
- [ ] Dùng `loop` thay cho việc lặp code thủ công.
- [ ] Dùng `when` xử lý đúng khác biệt hệ điều hành.
- [ ] Hiểu và chứng minh được bằng log: Handler chỉ chạy khi có thay đổi, và chỉ chạy một lần dù nhiều Task cùng notify.
- [ ] Viết được `block/rescue/always` xử lý lỗi có cấu trúc.
- [ ] Tổ chức Playbook lớn bằng `import_tasks`/`include_tasks` + `tags`, chạy chọn lọc bằng `--tags`.
