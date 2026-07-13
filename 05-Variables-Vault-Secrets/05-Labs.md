---
title: "05 - Labs"
module: 5
tags: [ansible, sysops-infra, module-05, labs]
---

# 05 — Labs thực hành

> Yêu cầu môi trường: control node + inventory/Playbook từ các module trước.

## Lab 1 — Thử nghiệm Variable Precedence thực tế

**Mục tiêu:** thấy tận mắt thứ tự ưu tiên biến hoạt động, không chỉ học lý thuyết.

**Các bước:**

1. Khai báo `app_env: "tu-group-vars"` trong `group_vars/webservers.yml`.
2. Khai báo `app_env: "tu-host-vars"` trong `host_vars/web-01.yml`.
3. Thêm `vars: { app_env: "tu-play-vars" }` trong Play của `site.yml`.
4. Chạy `ansible-playbook site.yml -m ansible.builtin.debug -a "var=app_env"` (hoặc thêm Task `debug` in biến này) — không dùng `--extra-vars`, ghi lại giá trị hiển thị.
5. Chạy lại với `ansible-playbook site.yml --extra-vars "app_env=tu-extra-vars"` — ghi lại giá trị hiển thị, xác nhận đúng theo bảng precedence.

**Kết quả cần có:** 2 log chạy khác nhau, đối chiếu với bảng precedence ở [[02-Theory|02 - Theory]] để giải thích vì sao mỗi lần ra giá trị đó.

## Lab 2 — Mã hóa toàn bộ file Vault

**Mục tiêu:** tạo, sửa, dùng một file Vault hoàn chỉnh trong Playbook.

**Các bước:**

1. Tạo `group_vars/production/vault.yml` bằng `ansible-vault create`, khai báo biến `db_password: "S3cretPassword123"`.
2. Xác nhận file trên đĩa là nội dung mã hóa (mở bằng `cat`, không đọc được), thử `ansible-vault view` để xem nội dung thật.
3. Viết một Task `debug` in ra biến `db_password` trong Playbook nhắm vào `production` group.
4. Chạy Playbook — trước tiên **không** cung cấp password Vault, quan sát lỗi. Sau đó chạy lại với `--ask-vault-pass`, xác nhận in đúng giá trị.

**Kết quả cần có:** file `vault.yml` đã mã hóa, log lỗi khi thiếu password, log thành công khi có password.

## Lab 3 — `encrypt_string` cho biến đơn lẻ

**Mục tiêu:** mã hóa chỉ 1 biến nhạy cảm trong file phần lớn plaintext.

**Các bước:**

1. Viết file `group_vars/production/vars.yml` (không mã hóa) với 2 biến plaintext (`app_name`, `app_port`).
2. Dùng `ansible-vault encrypt_string` tạo giá trị mã hóa cho biến thứ 3 (`api_key`), dán kết quả vào cuối file `vars.yml` như một biến `!vault`.
3. Chạy Playbook in cả 3 biến bằng `debug` — xác nhận `app_name`/`app_port` đọc được ngay (không cần password Vault để parse YAML), nhưng cần password để giải mã đúng `api_key`.

**Kết quả cần có:** file `vars.yml` có 2 biến plaintext + 1 biến `!vault`, log chạy xác nhận cả 3 hiển thị đúng khi có password.

## Lab 4 — Multiple Vault ID cho 2 môi trường

**Mục tiêu:** quản lý secret riêng biệt cho dev và production.

**Các bước:**

1. Tạo `group_vars/dev/vault.yml` mã hóa với `--vault-id dev@prompt`.
2. Tạo `group_vars/production/vault.yml` mã hóa với `--vault-id production@prompt` (dùng password khác với dev).
3. Viết Playbook có 2 Play, một nhắm `dev`, một nhắm `production`, mỗi Play in biến secret tương ứng.
4. Chạy `ansible-playbook site.yml --vault-id dev@prompt --vault-id production@prompt` — Ansible sẽ hỏi lần lượt 2 password khác nhau, xác nhận từng Play giải mã đúng bằng đúng password của môi trường đó.

**Kết quả cần có:** 2 file vault riêng biệt, log chạy xác nhận cả 2 môi trường giải mã đúng khi cung cấp đúng password tương ứng.

## Lab 5 — Rotate password Vault

**Mục tiêu:** thực hành xoay vòng password mã hóa định kỳ — một thao tác vận hành thực tế, không chỉ lý thuyết.

**Các bước:**

1. Chọn 1 file Vault đã tạo ở Lab 2, ghi lại password hiện tại.
2. Chạy `ansible-vault rekey` đổi sang password mới.
3. Xác nhận Playbook chạy với password **cũ** giờ báo lỗi, chạy với password **mới** thành công.
4. Ghi chú lại quy trình này như một runbook ngắn (dùng khi nào, ai cần được thông báo) — lưu vào `labs/` để tham khảo sau.

**Kết quả cần có:** log lỗi với password cũ, log thành công với password mới, ghi chú runbook ngắn.

## Checklist hoàn thành module

- [ ] Giải thích được (không tra bảng) kết quả precedence cho một tình huống biến khai báo ở 3+ nơi khác nhau.
- [ ] Tạo và dùng thành thạo file Vault mã hóa toàn bộ (`create`, `edit`, `view`).
- [ ] Dùng `encrypt_string` cho biến đơn lẻ trong file phần lớn plaintext.
- [ ] Quản lý được ít nhất 2 Vault ID cho 2 môi trường khác nhau trong cùng một Playbook.
- [ ] Thực hiện được `rekey` để xoay vòng password Vault.
