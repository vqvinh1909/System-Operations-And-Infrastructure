---
title: "04 - Commands"
module: 1
tags: [ansible, sysops-infra, module-01, commands, ssh, yaml]
---

# 04 — Lệnh & Cú pháp tham chiếu

> Đây là bảng tra cứu nhanh. Không cần học thuộc — mục tiêu là biết lệnh nào tồn tại để tra lại khi cần, và hiểu **vì sao** dùng ở bước nào trong quy trình chuẩn bị cho Ansible.

## 1. Tạo và triển khai SSH key cho automation

```bash
# Tao cap key rieng cho automation, khong dung chung voi key ca nhan
ssh-keygen -t ed25519 -f ~/.ssh/id_automation -C "ansible-automation-key"

# Copy public key sang managed node (dung mot lan, qua user co san password/key khac)
ssh-copy-id -i ~/.ssh/id_automation.pub deploy@10.0.1.11

# Kiem tra ket noi bang key moi, khong hoi password
ssh -i ~/.ssh/id_automation deploy@10.0.1.11 "hostname"
```

Vì sao `ed25519` thay vì `rsa`: key ngắn hơn, tốc độ ký nhanh hơn, được khuyến nghị hiện tại cho hạ tầng mới; `rsa 4096` vẫn dùng được nếu managed node là hệ thống cũ chưa hỗ trợ ed25519 (một số thiết bị mạng, OS cổ).

## 2. Quét fingerprint trước khi tự động hóa (tránh treo do known_hosts)

```bash
# Quet va them fingerprint cua nhieu host vao known_hosts truoc khi chay automation
ssh-keyscan -H 10.0.1.11 10.0.1.12 10.0.1.20 >> ~/.ssh/known_hosts
```

Vì sao cần bước này: nếu bỏ qua, lần đầu Ansible kết nối tới host mới sẽ dừng lại chờ xác nhận `yes/no` — hành vi tương tác này phá vỡ automation không người trực (CI/CD, cron).

## 3. Cấu hình `~/.ssh/config`

```
# ~/.ssh/config
Host web-*
    User deploy
    IdentityFile ~/.ssh/id_automation
    StrictHostKeyChecking accept-new

Host web-01
    HostName 10.0.1.11
```

Kiểm tra cấu hình đã đúng chưa mà không thật sự kết nối:

```bash
ssh -G web-01
```

Lệnh này in ra toàn bộ tham số SSH sẽ được áp dụng khi kết nối tới `web-01` (đã gộp từ khối `Host web-*` và `Host web-01`) — cách nhanh nhất để debug lỗi kế thừa cấu hình sai.

## 4. Kiểm tra quyền sudo của service account

```bash
# Kiem tra file sudoers co loi cu phap khong TRUOC KHI ap dung
visudo -cf /etc/sudoers.d/ansible

# Test quyen sudo cua user deploy tu xa, khong can biet password
ssh -i ~/.ssh/id_automation deploy@10.0.1.11 "sudo -n whoami"
```

`sudo -n` (non-interactive): nếu trả về `root` là NOPASSWD hoạt động đúng; nếu trả về lỗi `sudo: a password is required` nghĩa là cấu hình `/etc/sudoers.d/` chưa đúng — đây chính xác là lỗi Ansible sẽ gặp khi chạy task có `become: true`.

## 5. Kiểm tra cú pháp YAML trước khi dùng

```bash
# Cach 1: dung python co san (khong can cai them)
python3 -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))" inventory.yml

# Cach 2: dung yamllint (khuyen nghi, can cai: pip install yamllint)
yamllint inventory.yml
```

Thói quen nên tập ngay từ module này: chạy kiểm tra cú pháp YAML **trước khi** đưa file vào Ansible — lỗi YAML luôn dễ sửa hơn nhiều khi bị bắt sớm, thay vì lẫn trong log lỗi dài của Ansible ở các module sau.

## 6. Bảng cú pháp YAML nhanh

| Cấu trúc | Cú pháp | Ghi chú |
|---|---|---|
| Scalar | `key: value` | Số/chuỗi/boolean, dấu `:` theo sau bởi 1 space |
| Chuỗi có ký tự đặc biệt | `key: "gia tri: co dau hai cham"` | Bắt buộc để trong ngoặc kép nếu chuỗi chứa `:`, `{{ }}`, `#` |
| List | `- item` | Thụt lề cùng cấp, mỗi dòng bắt đầu bằng `- ` |
| Dictionary lồng | `parent:\n  child: value` | Thụt lề 2 space thể hiện quan hệ cha-con |
| Multi-document | `---` | Đánh dấu bắt đầu 1 document YAML (Ansible playbook luôn có) |
| Comment | `# ghi chu` | Không có cú pháp comment nhiều dòng |

## 7. Cú pháp Jinja2 nhận diện nhanh

| Ký hiệu | Ý nghĩa | Ví dụ |
|---|---|---|
| `{{ ... }}` | Xuất giá trị biểu thức | `{{ ansible_hostname }}` |
| `{% ... %}` | Câu lệnh điều khiển | `{% for pkg in packages %}` |
| `{# ... #}` | Comment, không xuất ra | `{# TODO: xem lai bien nay #}` |
| `{{ a | filter }}` | Áp dụng filter lên biến | `{{ name | upper }}` |
