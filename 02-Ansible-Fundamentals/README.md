---
title: "Module 02 - Ansible Fundamentals"
tags: [ansible, sysops-infra, module-02, fundamentals, inventory, ansible-cfg]
module: 2
part: "I - Infrastructure as Code (Ansible)"
difficulty: Foundation
status: draft
created: 2026-07-13
prerequisites: ["[[../01-Linux-for-Automation-Review/README|Module 01]]"]
next: "[[../03-Playbooks/README|Module 03]]"
---

# Module 02 — Ansible Fundamentals

> [!info] Vị trí trong giáo trình
> Module 02/07 của Part I. Đây là module bạn **gõ lệnh Ansible đầu tiên**. Sau Module 00 (tư duy IaC) và Module 01 (chuẩn bị SSH/sudo/YAML), module này dựng nền kiến trúc thật: control node, managed node, inventory, variables, facts, modules, ad-hoc command, và file cấu hình trung tâm `ansible.cfg`. Mọi module sau (Playbooks, Templates, Vault, Roles, Enterprise Automation) đều xây trên nền này.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Giới thiệu Ansible, lịch sử ngắn, vì sao chọn Ansible so với Puppet/Chef/Salt |
| [[02-Theory]] | Kiến trúc agentless, Control Node, Managed Node, Inventory (Static/Dynamic/Plugin/Cloud), Variables, Facts, Modules, ansible.cfg |
| [[03-Architecture]] | Sơ đồ kiến trúc tổng thể, luồng thực thi module, phân cấp inventory |
| [[04-Commands]] | Cài đặt, `ansible --version`, ad-hoc command, `ansible-inventory`, `ansible-doc` |
| [[05-Labs]] | Cài đặt control node, viết inventory tĩnh, chạy ad-hoc, khám phá facts |
| [[06-Troubleshooting]] | Lỗi UNREACHABLE, lỗi inventory parsing, lỗi Python interpreter, lỗi ansible.cfg precedence |
| [[07-Interview]] | Câu hỏi phỏng vấn kiến trúc Ansible, inventory, module |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 03 |

## Learning Objectives (tổng)

1. Giải thích kiến trúc agentless của Ansible: control node đẩy module qua SSH, managed node không cần cài daemon thường trực.
2. Cài đặt Ansible đúng cách trên control node, phân biệt `ansible-core` và gói `ansible` (community package đi kèm collections).
3. Viết được inventory tĩnh dạng INI và YAML, hiểu sự khác biệt giữa inventory tĩnh, dynamic inventory script, và inventory plugin (giới thiệu cloud AWS/Azure/GCP).
4. Phân biệt Variables (do người dùng định nghĩa) và Facts (Ansible tự thu thập từ managed node qua module `setup`).
5. Hiểu khái niệm Module trong Ansible — đơn vị thực thi nhỏ nhất, idempotent, trả về JSON.
6. Chạy thành thạo Ad-hoc Command cho các tác vụ một lần (ping, kiểm tra uptime, copy file nhanh) — phân biệt khi nào dùng ad-hoc, khi nào cần Playbook (Module 03).
7. Cấu hình `ansible.cfg` và hiểu thứ tự ưu tiên đọc cấu hình (environment variable > `ANSIBLE_CONFIG` > `./ansible.cfg` > `~/.ansible.cfg` > `/etc/ansible/ansible.cfg`).

## Self-Review

> [!note] Self-Review
> - **Accuracy:** Đã WebSearch xác nhận qua endoflife.date/ansible-core và Ansible Community Documentation (cập nhật 06/2026): ansible-core nhánh ổn định mới nhất là **2.21** (phát hành 18/05/2026, hỗ trợ tới 30/11/2027), nhánh 2.20 và 2.19 vẫn được hỗ trợ song song theo mô hình 3-release. Control node yêu cầu Python 3.12–3.14 với ansible-core 2.21; managed node hỗ trợ Python 3.9–3.14. Gói cộng đồng `ansible` (bundle collections) bản mới nhất là 14.1.0 (18/06/2026). Số liệu này được ghi rõ dưới dạng "tại thời điểm biên soạn" trong [[02-Theory]] để tránh lỗi thời khi phiên bản mới ra mắt.
> - **Diagram:** Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã đo bằng script Python (`len()` từng dòng) trước khi đưa vào file.
> - **Scope:** Dynamic Inventory/Cloud (AWS/Azure/GCP) chỉ giới thiệu khái niệm và cú pháp plugin ở mức nền tảng — triển khai chi tiết plugin cloud cụ thể không thuộc phạm vi khóa này (khóa tập trung Ansible core, không đi sâu tích hợp cloud provider).

## Điều hướng

- Quay lại: [[../01-Linux-for-Automation-Review/README|Module 01 — Linux for Automation Review]]
- Tiếp theo: [[../03-Playbooks/README|Module 03 — Playbooks]]
