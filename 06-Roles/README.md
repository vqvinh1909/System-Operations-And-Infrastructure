---
title: "Module 06 - Roles"
tags: [ansible, sysops-infra, module-06, roles, galaxy, collections]
module: 6
part: "I - Infrastructure as Code (Ansible)"
difficulty: Advanced
status: draft
created: 2026-07-13
prerequisites: ["[[../05-Variables-Vault-Secrets/README|Module 05]]"]
next: "[[../07-Enterprise-Automation/README|Module 07]]"
---

# Module 06 — Roles

> [!info] Vị trí trong giáo trình
> Module 06/07 của Part I — module cuối cùng trước khi ráp mọi thứ lại thành hệ thống doanh nghiệp ở Module 07. Tới đây, Playbook của bạn đã có Task, Template, Variables, Vault — nhưng vẫn nằm trong một file/dự án đơn lẻ. Role là cách Ansible chuẩn hóa việc **đóng gói** toàn bộ những thứ đó thành một đơn vị có cấu trúc cố định, tái sử dụng được giữa nhiều dự án, và chia sẻ được qua Ansible Galaxy/Collections — kỹ năng bắt buộc để làm việc trong bất kỳ team Ansible trưởng thành nào.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Vì sao copy-paste Task giữa các dự án là nợ kỹ thuật |
| [[02-Theory]] | Directory Structure chuẩn, Defaults vs Vars, Meta, Dependencies, Galaxy, Collections, Best Practices |
| [[03-Architecture]] | Sơ đồ cấu trúc thư mục Role, sơ đồ Role Dependencies, sơ đồ Galaxy/Collection namespace |
| [[04-Commands]] | `ansible-galaxy role/collection init/install/list`, cấu trúc `requirements.yml` |
| [[05-Labs]] | Tạo Role từ đầu, chuyển Playbook cũ thành Role, khai báo dependencies, cài Role từ Galaxy |
| [[06-Troubleshooting]] | Lỗi đường dẫn Role không tìm thấy, lỗi Role variable bị ghi đè sai, lỗi dependencies vòng lặp |
| [[07-Interview]] | Câu hỏi phỏng vấn Roles, Galaxy, Collections |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 07 |

## Learning Objectives (tổng)

1. Giải thích lý do Role tồn tại: tái sử dụng, chuẩn hóa, chia sẻ — không phải chỉ là "cách tổ chức thư mục cho đẹp".
2. Tạo đúng cấu trúc thư mục Role chuẩn bằng `ansible-galaxy role init`, hiểu vai trò của từng thư mục con (`tasks`, `handlers`, `templates`, `files`, `vars`, `defaults`, `meta`).
3. Phân biệt `defaults/main.yml` và `vars/main.yml` — hai nơi đều chứa biến nhưng khác hẳn về độ ưu tiên và mục đích sử dụng.
4. Khai báo Role Dependencies trong `meta/main.yml`, hiểu thứ tự thực thi khi một Role phụ thuộc Role khác.
5. Phân biệt Ansible Galaxy (kho Role/Collection cộng đồng) và Collection (đơn vị đóng gói module/plugin/role chính thức từ Ansible 2.10 trở đi).
6. Áp dụng Role Best Practices: một Role làm đúng một việc, tài liệu hóa qua `README.md`, khai báo biến mặc định hợp lý trong `defaults`.

## Self-Review

> [!note] Self-Review
> - **Accuracy:** Đã WebSearch xác nhận qua Ansible Community Documentation (07/2026): cấu trúc thư mục Role chuẩn không đổi qua nhiều năm (ổn định từ Ansible 2.x), mô hình Collection (namespace.collection_name) là cách đóng gói chính thức từ Ansible 2.10, `ansible-galaxy collection install` và `ansible-galaxy role install` vẫn là 2 lệnh tách biệt cho 2 loại nội dung khác nhau tính tới ansible-core 2.21.
> - **Diagram:** Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã đo bằng script Python trước khi đưa vào.
> - **Cấm:** [[05-Labs]] chỉ chứa nội dung kỹ thuật, không có câu hỏi phỏng vấn.

## Điều hướng

- Quay lại: [[../05-Variables-Vault-Secrets/README|Module 05 — Variables, Vault & Secrets]]
- Tiếp theo: [[../07-Enterprise-Automation/README|Module 07 — Enterprise Automation]]
