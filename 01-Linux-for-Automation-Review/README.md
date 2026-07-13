---
title: "Module 01 - Linux for Automation Review"
tags: [ansible, sysops-infra, module-01, linux-review, ssh, yaml, jinja2]
module: 1
part: "I - Infrastructure as Code (Ansible)"
difficulty: Foundation
status: draft
created: 2026-07-12
prerequisites: ["[[../00-Infrastructure-Automation-Fundamentals/README|Module 00]]"]
next: "[[../02-Ansible-Fundamentals/README|Module 02]]"
---

# Module 01 — Linux for Automation Review

> [!info] Vị trí trong giáo trình
> Module 01/07 của Part I. **Không dạy lại Linux từ đầu** — bạn đã học kỹ SSH, sudo, filesystem, systemd ở khóa Linux System Administrator (29 module). Module này ôn nhanh và định hướng lại đúng những phần Linux sẽ được Ansible sử dụng trực tiếp, đồng thời giới thiệu 2 kỹ năng mới cần cho automation: YAML và Jinja2 cơ bản.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Giới thiệu, phạm vi ôn tập |
| [[02-Theory]] | SSH/SSH Key/SSH Config dưới góc nhìn automation, Sudo cho service account, Inventory Planning, YAML Basics, Jinja2 Basics |
| [[03-Architecture]] | Sơ đồ luồng xác thực SSH key trong Ansible, sơ đồ phân nhóm inventory |
| [[04-Commands]] | Lệnh ssh-keygen, ssh-copy-id, cấu hình ~/.ssh/config, cú pháp YAML/Jinja2 mẫu |
| [[05-Labs]] | Lab tạo SSH key riêng cho automation, viết YAML inventory nháp, thử Jinja2 |
| [[06-Troubleshooting]] | Lỗi SSH key thường gặp khi chuẩn bị cho Ansible, lỗi cú pháp YAML |
| [[07-Interview]] | Câu hỏi phỏng vấn SSH/YAML/Jinja2 liên quan automation |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 02 |

## Learning Objectives (tổng)

1. Giải thích tại sao SSH key-based authentication là điều kiện bắt buộc (không phải tùy chọn) để vận hành Ansible ở quy mô doanh nghiệp.
2. Cấu hình `~/.ssh/config` để quản lý nhiều managed node với alias, user, key riêng — nền tảng cho Ansible inventory sau này.
3. Thiết kế mô hình sudo cho một "service account" chuyên dùng để automation, áp dụng nguyên tắc least privilege.
4. Lập kế hoạch (trên giấy, chưa cần Ansible) cách nhóm host theo môi trường/vai trò — chuẩn bị tư duy cho Inventory ở Module 02.
5. Đọc và viết đúng cú pháp YAML: scalar, list, dictionary, indentation, comment — vì **toàn bộ Ansible được viết bằng YAML**.
6. Đọc hiểu cú pháp Jinja2 cơ bản (`{{ }}`, `{% %}`) đủ để không bỡ ngỡ khi gặp trong Module 02-04.

## Self-Review

> [!note] Self-Review
> - **Accuracy:** Đã đối chiếu cú pháp SSH config (`~/.ssh/config`), YAML (indentation 2 space theo chuẩn Ansible), và Jinja2 delimiter (`{{ }}`, `{% %}`, `{# #}`) với tài liệu chính thức Ansible Community Documentation và Jinja2 documentation — không có thay đổi cú pháp nền tảng nào tính đến 2026-07-12.
> - **Scope:** Module này cố tình **ôn nhanh, không giảng lại từ số 0** — nội dung SSH/sudo chi tiết tham chiếu ngược về khóa Linux Sysadmin (Module 20 SSH Hardening, Module 09 User & Sudo) thay vì lặp lại.
> - **Diagram:** 2 sơ đồ ASCII trong [[03-Architecture]] đã đo bằng script Python trước khi đưa vào.

## Điều hướng

- Quay lại: [[../00-Infrastructure-Automation-Fundamentals/README|Module 00 — Infrastructure Automation Fundamentals]]
- Tiếp theo: [[../02-Ansible-Fundamentals/README|Module 02 — Ansible Fundamentals]]
