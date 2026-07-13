---
title: "08 - Summary"
module: 2
tags: [ansible, sysops-infra, module-02, summary]
---

# 08 — Tóm tắt Module 02

## Những gì đã học

- **Kiến trúc agentless:** Control Node đẩy module qua SSH, Managed Node không cần daemon thường trực — chỉ cần SSH server + Python.
- **Inventory:** tĩnh (INI/YAML), động (script), plugin (cloud AWS/Azure/GCP), một host thuộc nhiều nhóm cùng lúc.
- **Variables vs Facts:** Variables do người dùng khai báo, Facts do Ansible tự thu thập qua module `setup`.
- **Modules:** đơn vị thực thi nhỏ nhất, idempotent, ưu tiên module chuyên biệt hơn `command`/`shell`.
- **Ad-hoc Command:** dùng cho tác vụ một lần, khẩn cấp — chuyển sang Playbook khi cần lặp lại hoặc có logic phức tạp.
- **`ansible.cfg`:** thứ tự ưu tiên đọc cấu hình, best practice đặt trong thư mục dự án và commit Git.

## Bảng đối chiếu 4 viên gạch nền

| Khái niệm | Vai trò | Học sâu hơn ở |
|---|---|---|
| Control Node | Nơi chạy lệnh, lưu inventory/playbook | Toàn khóa |
| Managed Node | Máy bị quản lý, thực thi module | Toàn khóa |
| Inventory | Danh sách host + phân nhóm | Module 05 (group_vars/host_vars) |
| Module | Đơn vị thực thi idempotent | Module 03 (dùng trong Task) |

## Điều kiện để chuyển sang Module 03

1. Cài đặt và xác nhận Ansible hoạt động trên control node?
2. Viết được inventory YAML với nhóm cha/con, kiểm tra bằng `ansible-inventory --graph`?
3. Chạy được ad-hoc command với module chuyên biệt, quan sát và giải thích được `changed: true/false`?
4. Phân biệt rõ Variables và Facts, biết lệnh thu thập/lọc facts?
5. Có file `ansible.cfg` riêng cho dự án lab của bạn, hiểu vì sao đặt ở đó?

## Cầu nối sang Module 03

Từ Module 03, bạn sẽ **gộp nhiều ad-hoc command thành một Playbook** — file YAML mô tả chuỗi Task tuần tự, có biến, có điều kiện, có vòng lặp, có xử lý lỗi. Mọi khái niệm nền tảng ở Module 02 (Inventory, Module, Variables, Facts) đều được dùng lại nguyên vẹn bên trong Playbook — Module 03 không thay thế mà **xây thêm lớp tổ chức** lên trên những gì bạn vừa học.

## Điều hướng

- Quay lại: [[README|Module 02 — README]]
- Tiếp theo: [[../03-Playbooks/README|Module 03 — Playbooks]]
