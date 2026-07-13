---
title: "04 - Commands"
module: 5
tags: [ansible, sysops-infra, module-05, commands, ansible-vault]
---

# 04 — Lệnh & Cú pháp tham chiếu

## 1. Quản lý file mã hóa toàn bộ

```bash
# Tao moi file da ma hoa tu dau
ansible-vault create group_vars/production/vault.yml

# Ma hoa file YAML plaintext co san
ansible-vault encrypt group_vars/production/secrets.yml

# Sua noi dung (tu dong giai ma - mo editor - ma hoa lai khi luu)
ansible-vault edit group_vars/production/vault.yml

# Xem noi dung, khong sua
ansible-vault view group_vars/production/vault.yml

# Giai ma vinh vien (can than - chi dung tam thoi de migrate)
ansible-vault decrypt group_vars/production/vault.yml

# Doi password ma hoa (rotate), khong doi noi dung ben trong
ansible-vault rekey group_vars/production/vault.yml
```

## 2. Mã hóa biến đơn lẻ

```bash
# Ma hoa 1 gia tri, dan ket qua vao file YAML thuong
ansible-vault encrypt_string 'S3cretPassword123' --name 'db_password'

# Doc gia tri can ma hoa tu file thay vi go tay (tranh luu trong bash history)
ansible-vault encrypt_string --name 'db_password' < password.txt
```

## 3. Chạy Playbook với Vault

```bash
# Hoi password tuong tac
ansible-playbook site.yml --ask-vault-pass

# Doc password tu file (quyen file phai la 600)
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt

# Dung Vault ID co nhan, ho tro nhieu vault cung luc
ansible-playbook site.yml --vault-id production@~/.vault_pass_prod.txt

# Nhieu vault id cung luc (vi du can ca secret dev va secret shared)
ansible-playbook site.yml --vault-id dev@prompt --vault-id shared@~/.vault_pass_shared.txt
```

## 4. Cấu hình mặc định trong `ansible.cfg` (tránh gõ lại mỗi lần)

```ini
[defaults]
vault_identity_list = production@~/.vault_pass_prod.txt, dev@prompt
```

## 5. Debug Variable Precedence

```bash
# Xem gia tri cuoi cung cua 1 bien tren 1 host, da gop tu moi nguon
ansible web-01 -m ansible.builtin.debug -a "var=http_port"

# Xem toan bo bien da resolve cho 1 host (bao gom ca facts)
ansible-inventory -i inventory.yml --host web-01

# Chay voi -vvvv de xem Ansible dang doc bien tu file/nguon nao (rat chi tiet)
ansible-playbook site.yml -vvvv --check
```

## 6. Bảng tổng hợp lệnh `ansible-vault`

| Lệnh | Chức năng |
|---|---|
| `create` | Tạo file mới, mã hóa ngay từ đầu |
| `encrypt` | Mã hóa file plaintext có sẵn |
| `decrypt` | Giải mã vĩnh viễn |
| `edit` | Sửa nội dung (tự giải mã/mã hóa lại) |
| `view` | Xem nội dung, không sửa |
| `rekey` | Đổi password mã hóa |
| `encrypt_string` | Mã hóa một giá trị đơn lẻ |

## 7. Lưu ý an toàn khi thao tác dòng lệnh

```bash
# TRANH: go password truc tiep vao lenh, se luu trong bash history
ansible-vault encrypt_string 'S3cret' --name 'db_password'   # van con S3cret trong history

# TOT HON: dung stdin, khong de lo trong history
echo -n 'S3cret' | ansible-vault encrypt_string --stdin-name 'db_password'

# Dat quyen file password Vault cuc bo dung 600
chmod 600 ~/.vault_pass.txt
```
