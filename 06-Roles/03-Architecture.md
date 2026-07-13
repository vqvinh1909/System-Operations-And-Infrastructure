---
title: "03 - Architecture"
module: 6
tags: [ansible, sysops-infra, module-06, architecture, diagram, roles, galaxy]
---

# 03 — Kiến trúc & Sơ đồ

## Sơ đồ 1: Playbook gọi Role — cấu trúc bên trong

```
+----------------------------+
|    site.yml (Playbook)     |
+----------------------------+
              |
              v
+--------------------------------------------+
| roles: [nginx_hardened, monitoring_agent]  |
+--------------------------------------------+
              |
              v
+----------------------------+
| Role nginx_hardened/       |
+----------------------------+
              |
              v
+--------------------------------------------+
|       tasks/  handlers/  templates/        |
|      files/  vars/  defaults/  meta/       |
+--------------------------------------------+
```

Đọc sơ đồ: `site.yml` chỉ cần khai báo **tên** Role muốn dùng — toàn bộ logic chi tiết (Task, Handler, Template, biến) nằm gọn trong thư mục Role tương ứng, không lộ ra ở cấp Playbook chính. Đây là điểm khác biệt căn bản so với Module 03, nơi mọi Task nằm trực tiếp trong file Playbook.

## Sơ đồ 2: Thứ tự thực thi khi Role có Dependencies

```
+--------------------------------------+
|       Goi role: nginx_hardened       |
+--------------------------------------+
                    |
                    v
+----------------------------------------------+
|   Buoc 1: Chay dependency common_hardening   |
+----------------------------------------------+
                    |
                    v
+----------------------------------------------+
|    Buoc 2: Chay dependency firewall_setup    |
+----------------------------------------------+
                    |
                    v
+------------------------------------------------------+
| Buoc 3: Chay chinh tasks/main.yml cua nginx_hardened |
+------------------------------------------------------+
```

Đọc sơ đồ: Dependencies khai báo trong `meta/main.yml` luôn chạy **trước** Task chính của Role, theo đúng thứ tự khai báo trong danh sách `dependencies`. Nếu nhiều Role khác nhau trong cùng Playbook đều phụ thuộc `common_hardening`, Ansible mặc định chỉ chạy Role đó **một lần duy nhất** — tránh lãng phí thời gian chạy lặp lại cùng một logic.

## Sơ đồ 3: Role vs Collection — phạm vi đóng gói

```
+--------------------------------------+
|     ROLE: geerlingguy.postgresql     |
|      (1 chuc nang dong goi san)      |
+--------------------------------------+

+--------------------------------------+
|        COLLECTION: amazon.aws        |
|    (nhieu Module + Plugin + Role)    |
+--------------------------------------+
```

Đọc sơ đồ: một Role đóng gói **một chức năng cụ thể**. Một Collection có phạm vi rộng hơn nhiều, có thể chứa **nhiều Role, Module, Plugin, Filter, Lookup** cùng lúc dưới một namespace thống nhất (`<tổ_chức>.<tên>`) — đây là lý do các Inventory Plugin cloud giới thiệu ở Module 02 (`amazon.aws.aws_ec2`) và module `ansible.posix.synchronize` ở Module 04 đều thuộc về Collection, không phải Role đơn lẻ.
