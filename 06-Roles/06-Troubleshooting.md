---
title: "06 - Troubleshooting"
module: 6
tags: [ansible, sysops-infra, module-06, troubleshooting, roles]
---

# 06 — Troubleshooting

## Lỗi 1: `ERROR! the role 'nginx_hardened' was not found`

**Triệu chứng:** Playbook báo không tìm thấy Role dù thư mục Role thật sự tồn tại.

**Nguyên nhân thường gặp:**
- Playbook không nằm cùng cấp với thư mục `roles/` theo quy ước Ansible tự tìm kiếm (Ansible tìm `roles/` tương đối theo vị trí Playbook, hoặc theo `roles_path` khai báo trong `ansible.cfg`).
- Gõ sai tên Role (phân biệt hoa/thường, hoặc dùng gạch dưới `_` thay vì gạch ngang `-` không nhất quán với tên thư mục thật).

**Cách xử lý:**
```ini
# ansible.cfg - khai bao ro roles_path neu cau truc du an khong theo mac dinh
[defaults]
roles_path = ./roles
```
```bash
# Xac nhan Ansible dang tim role o dau
ansible-config dump --only-changed | grep -i roles_path
```

## Lỗi 2: Biến Role bị ghi đè "không đúng như mong đợi"

**Triệu chứng:** khai báo `vars:` khi gọi Role nhưng giá trị bên trong Role vẫn không đổi.

**Nguyên nhân:** biến đang bị ghi đè trong `vars/main.yml` của chính Role đó (độ ưu tiên cấp 15, cao hơn `vars:` khai báo khi gọi Role ở cấp Play — thực chất là cấp Role params, số 20, thường cao hơn — cần xem chính xác đường nào của biến này. Trường hợp phổ biến nhất: nhầm đặt biến cần cho người dùng tùy chỉnh vào `vars/main.yml` thay vì `defaults/main.yml`).

**Cách xử lý:** rà lại đúng nguyên tắc Module 05: **chỉ những biến trong `defaults/main.yml` mới thực sự "dễ ghi đè"** theo mọi cách thông thường (group_vars, host_vars, vars khi gọi role). Nếu biến nằm trong `vars/main.yml` của Role, nó có độ ưu tiên cao và không nên được thiết kế để người dùng ghi đè — đây là vấn đề thiết kế Role, không phải lỗi cú pháp.

## Lỗi 3: Role Dependencies chạy vòng lặp (circular dependency)

**Triệu chứng:** `ansible-playbook` treo hoặc báo lỗi liên quan tới dependencies khi Role A phụ thuộc Role B, Role B lại phụ thuộc Role A (trực tiếp hoặc gián tiếp qua Role C).

**Nguyên nhân:** thiết kế `meta/main.yml` tạo vòng lặp phụ thuộc — lỗi kiến trúc, không phải lỗi cú pháp.

**Cách xử lý:** vẽ lại sơ đồ phụ thuộc giữa các Role trên giấy trước khi khai báo `dependencies` — nếu phát hiện có chu trình khép kín, cần tách phần logic dùng chung ra một Role thứ ba độc lập mà cả A và B cùng phụ thuộc (thay vì phụ thuộc lẫn nhau).

## Lỗi 4: Cài Role từ Galaxy nhưng Playbook vẫn dùng bản cũ

**Triệu chứng:** đã chạy `ansible-galaxy role install <name>,<version-moi>` nhưng hành vi Playbook không đổi.

**Nguyên nhân:** `ansible-galaxy role install` **không tự động ghi đè** Role đã cài sẵn theo mặc định — cần cờ `--force` để cài đè lên bản cũ.

**Cách xử lý:**
```bash
ansible-galaxy role install geerlingguy.postgresql,4.0.0 --force
```

## Lỗi 5: Role hoạt động đúng khi test độc lập nhưng lỗi khi ráp vào Playbook lớn

**Triệu chứng:** `test-nginx-role.yml` (Lab 5) chạy hoàn hảo, nhưng khi gọi cùng Role đó trong `site.yml` (Playbook lớn có nhiều Role khác) thì lỗi.

**Nguyên nhân thường gặp:** xung đột tên biến giữa các Role (một Role khác cũng dùng tên biến `port` không có tiền tố, vô tình ghi đè lẫn nhau) — đúng lý do Best Practices ở [[02-Theory|02 - Theory]] mục 7 nhấn mạnh đặt tên biến có tiền tố tên Role.

**Cách xử lý:** rà soát toàn bộ tên biến trong các Role đang dùng chung Playbook, đảm bảo mọi biến đều có tiền tố rõ ràng (`nginx_port`, không phải `port`) để loại trừ khả năng xung đột.
