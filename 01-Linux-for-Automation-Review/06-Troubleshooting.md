---
title: "06 - Troubleshooting"
module: 1
tags: [ansible, sysops-infra, module-01, troubleshooting, ssh, yaml]
---

# 06 — Troubleshooting

## Lỗi 1: `Permission denied (publickey)`

**Triệu chứng:** `ssh -i ~/.ssh/id_automation deploy@10.0.1.11` báo `Permission denied (publickey)`.

**Nguyên nhân thường gặp:**
- Public key chưa nằm trong `~/.ssh/authorized_keys` của user `deploy` trên managed node.
- Sai quyền thư mục/file: `~/.ssh` phải là `700`, `authorized_keys` phải là `600` — SSH server từ chối key nếu quyền quá lỏng lẻo (nhóm/other có quyền ghi).
- Chỉ định sai key (`-i` trỏ nhầm file, hoặc quên `-i` nên dùng key mặc định `id_rsa`).

**Cách xử lý:**
```bash
# Kiem tra quyen tren managed node
ssh existing-user@10.0.1.11 "ls -la ~deploy/.ssh"
# Sua quyen neu sai
ssh existing-user@10.0.1.11 "chmod 700 ~deploy/.ssh && chmod 600 ~deploy/.ssh/authorized_keys"
# Debug chi tiet qua trinh SSH thu key nao
ssh -vvv -i ~/.ssh/id_automation deploy@10.0.1.11
```
Cờ `-vvv` cho thấy chính xác SSH client đang thử key nào, server từ chối ở bước nào — kỹ năng debug này sẽ dùng lại nguyên vẹn khi Ansible báo `UNREACHABLE` ở Module 02.

## Lỗi 2: Ansible/script tự động bị treo (hang) ở lần kết nối đầu tiên

**Triệu chứng:** script/automation không phản hồi, không báo lỗi, không tiếp tục chạy.

**Nguyên nhân:** SSH đang chờ người dùng gõ `yes` để xác nhận fingerprint host mới (chưa có trong `known_hosts`) — nhưng không có ai ngồi trước màn hình để gõ.

**Cách xử lý:**
```bash
# Cach dung: quet truoc va them vao known_hosts
ssh-keyscan -H 10.0.1.11 >> ~/.ssh/known_hosts

# Cach chi dung cho lab/test, KHONG dung production:
export ANSIBLE_HOST_KEY_CHECKING=False
```
Tắt hoàn toàn `StrictHostKeyChecking` ở production là rủi ro bảo mật thật (mất khả năng phát hiện man-in-the-middle) — chỉ chấp nhận được trong lab cá nhân, không mang thói quen này vào môi trường doanh nghiệp.

## Lỗi 3: `sudo: a password is required` dù đã cấu hình NOPASSWD

**Triệu chứng:** `sudo -n whoami` báo lỗi cần password dù đã thêm dòng `NOPASSWD` vào `/etc/sudoers.d/ansible`.

**Nguyên nhân thường gặp:**
- File `/etc/sudoers.d/ansible` có lỗi cú pháp, bị `sudo` bỏ qua toàn bộ (không báo lỗi rõ ràng khi chạy sudo thường, chỉ phát hiện qua `visudo -cf`).
- Có một dòng cấu hình sudoers **khác** (ví dụ trong `/etc/sudoers` gốc, hoặc file khác trong `sudoers.d/` được đọc sau) ghi đè, vì sudoers áp dụng theo **thứ tự đọc file, dòng sau override dòng trước** cho cùng user/lệnh.
- Quên username hoặc gõ sai tên user trong file cấu hình.

**Cách xử lý:**
```bash
# Luon kiem tra cu phap TRUOC khi tin tuong file da dung
visudo -cf /etc/sudoers.d/ansible

# Xem tat ca quyen sudo hien tai cua user deploy (gop tu moi file)
sudo -l -U deploy
```

## Lỗi 4: YAML "trông đúng" nhưng parser báo lỗi

**Triệu chứng:** `yaml.safe_load()` hoặc `yamllint` báo lỗi dù mắt thường không thấy gì sai.

**Nguyên nhân phổ biến nhất:**
- **Trộn tab và space** để thụt lề — nhìn giống nhau trên màn hình nhưng YAML coi là khác nhau, thường báo lỗi khó hiểu kiểu `mapping values are not allowed here`.
- Thụt lề lệch 1 space giữa các phần tử cùng cấp trong 1 list/dictionary — không phải lúc nào cũng báo lỗi, đôi khi âm thầm đổi cấu trúc cha-con (nguy hiểm hơn báo lỗi rõ ràng).
- Dùng dấu `:` trong một chuỗi mà quên đặt trong ngoặc kép, ví dụ `msg: Ket noi that bai: timeout` — YAML hiểu nhầm phần sau dấu `:` thứ hai là bắt đầu một key mới.

**Cách xử lý:**
```bash
# Bat editor hien thi ky tu tab/space (vi du VSCode: "Render Whitespace": all)
# Kiem tra nhanh bang script
python3 -c "import yaml; print(yaml.safe_load(open('file.yml')))"
```
Nếu lệnh trên chạy được và in ra đúng cấu trúc mong đợi (dict lồng list đúng như dự tính), YAML hợp lệ — nếu output khác kỳ vọng dù không báo lỗi, đây chính là kiểu lỗi "thụt lề lệch âm thầm đổi cấu trúc" nói trên.

## Lỗi 5: Nhầm `{{ }}` Jinja2 với `{ }` YAML flow-mapping

**Triệu chứng:** lỗi `while parsing a flow mapping` khi viết `msg: {{ ten_bien }}` không có ngoặc kép.

**Nguyên nhân:** YAML có cú pháp riêng dùng `{ key: value }` để viết dictionary trên một dòng (flow style). Khi parser gặp `{{ ` mà không có ngoặc kép bao ngoài, nó cố hiểu đây là mở một flow-mapping YAML, không phải cú pháp Jinja2 — và thất bại vì cú pháp bên trong không hợp lệ theo flow-mapping.

**Cách xử lý:** luôn đặt biểu thức Jinja2 trong ngoặc kép khi dùng trong YAML:
```yaml
# Sai
msg: {{ ten_bien }}

# Dung
msg: "{{ ten_bien }}"
```
