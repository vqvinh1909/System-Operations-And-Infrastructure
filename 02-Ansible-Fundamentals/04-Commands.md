---
title: "04 - Commands"
module: 2
tags: [ansible, sysops-infra, module-02, commands, ad-hoc, inventory]
---

# 04 — Lệnh & Cú pháp tham chiếu

## 1. Cài đặt Ansible trên Control Node

```bash
# Debian/Ubuntu - qua pip (khuyen nghi de kiem soat version)
python3 -m pip install --user ansible

# RHEL/Rocky/AlmaLinux - qua package manager
sudo dnf install ansible-core

# Kiem tra sau khi cai
ansible --version
```

`ansible --version` in ra phiên bản `ansible-core`, phiên bản Python đang dùng, và đường dẫn file `ansible.cfg` đang được áp dụng (hữu ích để debug thứ tự ưu tiên cấu hình đã học ở [[03-Architecture|03 - Architecture]]).

## 2. Ad-hoc Command — cú pháp và ví dụ

```bash
# Kiem tra ket noi toi toan bo host trong inventory
ansible all -m ansible.builtin.ping

# Kiem tra ket noi toi mot nhom cu the
ansible webservers -m ansible.builtin.ping

# Chay lenh shell truc tiep (uu tien module chuyen biet khi co the)
ansible all -m ansible.builtin.command -a "uptime"

# Cai package (idempotent - module apt tu kiem tra truoc khi cai)
ansible webservers -b -m ansible.builtin.apt -a "name=nginx state=present"

# Restart service tren toan bo fleet - vi du tinh huong khan cap
ansible webservers -b -m ansible.builtin.systemd -a "name=nginx state=restarted"

# Copy file nhanh, mot lan
ansible webservers -m ansible.builtin.copy -a "src=./motd dest=/etc/motd"
```

Cờ `-b` (viết tắt `--become`) yêu cầu chạy với quyền sudo — tương ứng `become: true` sẽ dùng trong Playbook ở Module 03.

## 3. Kiểm tra Inventory

```bash
# Liet ke toan bo host va nhom, dang cay
ansible-inventory -i inventory.yml --graph

# Xuat toan bo inventory da resolve ra JSON (huu ich khi debug dynamic inventory/plugin)
ansible-inventory -i inventory.yml --list

# Chi liet ke host thuoc mot nhom cu the, khong ket noi
ansible webservers --list-hosts -i inventory.yml
```

## 4. Tra cứu tài liệu module ngay trong terminal

```bash
# Xem tai lieu day du cua mot module, kem vi du
ansible-doc ansible.builtin.apt

# Liet ke toan bo module co san trong mot collection
ansible-doc -l ansible.builtin | head -30

# Tim nhanh module theo tu khoa
ansible-doc -l | grep -i mysql
```

`ansible-doc` là nguồn tra cứu chính xác nhất — luôn ưu tiên hơn tìm kiếm Google vì nội dung khớp đúng phiên bản Ansible đang cài trên máy bạn.

## 5. Facts — thu thập và lọc

```bash
# Xem toan bo facts cua mot host
ansible web-01 -m ansible.builtin.setup

# Loc facts theo tu khoa (vi du chi xem thong tin mang)
ansible web-01 -m ansible.builtin.setup -a "filter=ansible_default_ipv4"

# Chi dinh ro Python interpreter tren managed node (huu ich khi server co nhieu ban Python)
ansible web-01 -m ansible.builtin.ping --extra-vars "ansible_python_interpreter=/usr/bin/python3"
# Luu y: "gather_facts" la tuy chon cap Playbook (gather_facts: false), khong ap dung cho
# ad-hoc command o day - lenh -m ping/-m setup ad-hoc khong tu dong gather facts nhu playbook.
```

## 6. Kiểm tra cấu hình đang áp dụng

```bash
# Xem toan bo gia tri cau hinh hien tai (da gop tu ansible.cfg + default)
ansible-config dump

# Chi xem gia tri khac mac dinh (de debug nhanh)
ansible-config dump --only-changed

# Xem file ansible.cfg nao dang duoc doc
ansible-config view
```

## 7. Bảng tổng hợp lệnh CLI chính của bộ Ansible

| Lệnh | Chức năng |
|---|---|
| `ansible` | Chạy ad-hoc command |
| `ansible-playbook` | Chạy playbook (Module 03) |
| `ansible-inventory` | Kiểm tra/xuất inventory |
| `ansible-doc` | Tra cứu tài liệu module |
| `ansible-config` | Kiểm tra cấu hình đang áp dụng |
| `ansible-galaxy` | Quản lý Role/Collection (Module 06) |
| `ansible-vault` | Mã hóa secret (Module 05) |
