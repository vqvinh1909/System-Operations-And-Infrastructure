---
title: "04 - Commands"
module: 6
tags: [ansible, sysops-infra, module-06, commands, ansible-galaxy]
---

# 04 — Lệnh & Cú pháp tham chiếu

## 1. Tạo Role mới đúng cấu trúc chuẩn

```bash
# Tao khung thu muc Role chuan, tu dong sinh moi file/thu muc can thiet
ansible-galaxy role init nginx_hardened

# Cau truc tao ra:
# nginx_hardened/{tasks,handlers,templates,files,vars,defaults,meta,tests}/
```

## 2. Cài Role/Collection từ Ansible Galaxy

```bash
# Cai mot role cu the
ansible-galaxy role install geerlingguy.postgresql

# Cai dung version cu the (khuyen nghi cho production)
ansible-galaxy role install geerlingguy.postgresql,3.5.0

# Cai mot collection
ansible-galaxy collection install community.general

# Cai toan bo dependencies khai bao trong requirements.yml
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

## 3. Quản lý Role/Collection đã cài

```bash
# Liet ke role da cai
ansible-galaxy role list

# Liet ke collection da cai
ansible-galaxy collection list

# Xoa role
ansible-galaxy role remove geerlingguy.postgresql
```

## 4. Cấu trúc `requirements.yml` chuẩn

```yaml
---
roles:
  - name: geerlingguy.postgresql
    version: "3.5.0"
  - name: geerlingguy.nginx
    version: "3.1.4"

collections:
  - name: community.general
    version: ">=8.0.0"
  - name: amazon.aws
    version: ">=7.0.0"
```

## 5. Gọi Role trong Playbook

```yaml
# Cach 1 - danh sach ten don gian
- hosts: webservers
  roles:
    - nginx_hardened
    - monitoring_agent

# Cach 2 - kem tham so tuy chinh, ghi de defaults
- hosts: webservers
  roles:
    - role: nginx_hardened
      vars:
        nginx_worker_connections: 2048
      tags: [web]

# Cach 3 - goi Role nhu mot Task (linh hoat hon, co the dat giua cac task khac)
- hosts: webservers
  tasks:
    - name: Chay role nginx_hardened giua chung cac task khac
      ansible.builtin.include_role:
        name: nginx_hardened
```

## 6. Kiểm tra Role trước khi dùng

```bash
# Kiem tra cu phap toan bo Playbook + Role no goi
ansible-playbook site.yml --syntax-check

# Chi liet ke Task se chay (bao gom Task ben trong Role), khong thuc thi
ansible-playbook site.yml --list-tasks
```

## 7. Bảng tổng hợp lệnh `ansible-galaxy`

| Lệnh | Chức năng |
|---|---|
| `role init <name>` | Tạo khung Role mới |
| `role install <name>` | Cài Role từ Galaxy |
| `role list` | Liệt kê Role đã cài |
| `collection install <name>` | Cài Collection từ Galaxy |
| `collection list` | Liệt kê Collection đã cài |
| `install -r requirements.yml` | Cài hàng loạt theo file khai báo |
