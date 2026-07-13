---
title: "Module 04 - Templates"
tags: [ansible, sysops-infra, module-04, templates, jinja2, filters, lookup]
module: 4
part: "I - Infrastructure as Code (Ansible)"
difficulty: Intermediate
status: draft
created: 2026-07-13
prerequisites: ["[[../03-Playbooks/README|Module 03]]"]
next: "[[../05-Variables-Vault-Secrets/README|Module 05]]"
---

# Module 04 — Templates

> [!info] Vị trí trong giáo trình
> Module 04/07 của Part I. Module 03 dùng `copy` để chép file cấu hình **tĩnh, giống hệt nhau** trên mọi host. Thực tế doanh nghiệp gần như không bao giờ như vậy — mỗi host thường cần IP, port, tên miền, giá trị tuning riêng. Module này dạy Jinja2 Templates sâu hơn Module 01 (vốn chỉ giới thiệu cú pháp), cùng các module xử lý file liên quan: `template`, `file`, `copy`, `fetch`, `synchronize`, và Filters/Lookup — bộ công cụ biến một file cấu hình thành file "sống", tự thích nghi theo từng host.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Vì sao copy file tĩnh không đủ cho hạ tầng thật |
| [[02-Theory]] | Jinja2 nâng cao, module `template`, Filters, Lookup, `file`/`copy`/`fetch`/`synchronize` |
| [[03-Architecture]] | Sơ đồ luồng render template, sơ đồ khác biệt copy vs template vs fetch vs synchronize |
| [[04-Commands]] | Cú pháp module `template`, filter thường dùng, lookup plugin, `--diff --check` với template |
| [[05-Labs]] | Viết template nginx/config động theo host, dùng filter, lookup file/env |
| [[06-Troubleshooting]] | Lỗi Jinja2 template, lỗi filter sai kiểu dữ liệu, lỗi quyền file sau render |
| [[07-Interview]] | Câu hỏi phỏng vấn Jinja2/Template/Filter |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 05 |

## Learning Objectives (tổng)

1. Viết file `.j2` (Jinja2 template) chứa biến, điều kiện (`{% if %}`), vòng lặp (`{% for %}`) để render cấu hình động theo từng host.
2. Dùng module `ansible.builtin.template` để render và triển khai file `.j2`, phân biệt rõ với `copy` (không render Jinja2).
3. Sử dụng thành thạo các Filter phổ biến (`default`, `upper`/`lower`, `join`, `to_json`, `regex_replace`...) để biến đổi giá trị biến ngay trong template/playbook.
4. Dùng Lookup Plugin (`file`, `env`, `pipe`) để lấy dữ liệu từ nguồn bên ngoài Ansible (file cục bộ, biến môi trường, output lệnh) vào biến.
5. Phân biệt 4 module xử lý file: `file` (quản lý thuộc tính file/thư mục), `copy` (chép nguyên văn), `fetch` (lấy file từ managed node về control node — chiều ngược lại), `synchronize` (đồng bộ thư mục qua rsync).
6. Debug lỗi Jinja2 template hiệu quả bằng `--check --diff` và thông báo lỗi cú pháp Jinja2.

## Self-Review

> [!note] Self-Review
> - **Accuracy:** Đã WebSearch đối chiếu Jinja2 documentation và Ansible Community Documentation (07/2026) về danh sách filter built-in và hành vi module `template`/`copy`/`fetch`/`synchronize` — không có thay đổi cú pháp/hành vi cốt lõi so với các bản ansible-core gần đây. Module này **không đề cập Ansible Vault** — mã hóa biến/secret được để lại toàn bộ cho [[../05-Variables-Vault-Secrets/README|Module 05]] theo đúng phạm vi đã phân chia.
> - **Diagram:** Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã đo bằng script Python trước khi đưa vào.
> - **Cấm:** [[05-Labs]] chỉ chứa nội dung kỹ thuật, không có câu hỏi phỏng vấn.

## Điều hướng

- Quay lại: [[../03-Playbooks/README|Module 03 — Playbooks]]
- Tiếp theo: [[../05-Variables-Vault-Secrets/README|Module 05 — Variables, Vault & Secrets]]
