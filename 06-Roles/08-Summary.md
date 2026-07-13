---
title: "08 - Summary"
module: 6
tags: [ansible, sysops-infra, module-06, summary]
---

# 08 — Tóm tắt Module 06

## Những gì đã học

- **Cấu trúc Role chuẩn:** `tasks`, `handlers`, `templates`, `files`, `vars`, `defaults`, `meta` — tự động nhận diện qua quy ước tên thư mục.
- **`defaults` vs `vars`:** defaults thấp nhất (dễ ghi đè), vars cao (nội bộ, không nên ghi đè).
- **Dependencies:** khai báo trong `meta/main.yml`, tự động chạy trước, tự loại trùng.
- **Ansible Galaxy:** kho Role/Collection cộng đồng, cài qua `ansible-galaxy role/collection install`, khai báo hàng loạt qua `requirements.yml`, luôn ghim version cho production.
- **Collections:** phạm vi rộng hơn Role, đóng gói Module/Plugin/Filter/Lookup/Role dưới namespace thống nhất.
- **Best Practices:** một Role một việc, tiền tố tên biến, tài liệu hóa qua README, test độc lập trước khi ráp hệ thống lớn.

## Bảng đối chiếu toàn Part I — nơi mỗi kỹ năng "hạ cánh" trong Role

| Kỹ năng học ở Module | Nằm ở đâu trong Role |
|---|---|
| Module (Module 02) | Gọi bên trong `tasks/main.yml` |
| Task/Loop/Conditional/Block/Handler (Module 03) | `tasks/main.yml`, `handlers/main.yml` |
| Template/Filter/Lookup (Module 04) | `templates/*.j2`, tham chiếu trong Task |
| Variables/Vault (Module 05) | `defaults/main.yml`, `vars/main.yml`, file Vault trong `vars/` |

## Điều kiện để chuyển sang Module 07

1. Tạo được Role đúng cấu trúc chuẩn, chuyển đổi thành công một Playbook cũ thành Role?
2. Phân biệt chính xác khi nào dùng `defaults`, khi nào dùng `vars`?
3. Khai báo và giải thích được cơ chế Role Dependencies?
4. Cài và tích hợp được một Role/Collection từ Ansible Galaxy vào dự án thật?

## Cầu nối sang Module 07

Module 07 — Enterprise Automation là module tổng hợp cuối cùng của Part I: bạn sẽ ráp các Role vừa học cách xây dựng thành một **hệ thống triển khai doanh nghiệp hoàn chỉnh** — Multi-tier Deployment (web/app/database nhiều tầng), Rolling Update và Zero Downtime (dùng `block/rescue` từ Module 03 và Handler để tối ưu), HAProxy/Nginx làm load balancer, và tích hợp Backup/Monitoring. Đây là module "thi cuối kỳ" của Part I, trước khi Part II (Docker, agent khác xây dựng song song) tiếp nối sang Module 08.

## Điều hướng

- Quay lại: [[README|Module 06 — README]]
- Tiếp theo: [[../07-Enterprise-Automation/README|Module 07 — Enterprise Automation]]
