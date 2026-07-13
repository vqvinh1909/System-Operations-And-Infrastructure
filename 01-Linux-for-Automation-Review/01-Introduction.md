---
title: "01 - Introduction"
module: 1
tags: [ansible, sysops-infra, module-01, introduction]
---

# 01 — Giới thiệu

## Vì sao cần một module "ôn tập"?

Bạn đã học SSH, sudo, filesystem, systemd rất kỹ ở khóa Linux System Administrator. Nhưng có một khoảng cách giữa "biết dùng SSH để tự mình login vào server" và "cấu hình SSH đúng cách để một công cụ tự động (Ansible) login vào hàng chục server thay bạn, không cần ai gõ password". Khoảng cách đó chính là nội dung module này.

Đây không phải bài học Linux mới — đây là bài học "nhìn lại kiến thức cũ dưới góc nhìn mới". Ba câu hỏi xuyên suốt module:

1. Cấu hình SSH của bạn hiện tại (từ khóa Linux Sysadmin) đã đủ để Ansible dùng chưa, hay cần chỉnh gì thêm?
2. Sudo cho một user thật (con người) và sudo cho một "service account" chạy automation có nên giống nhau không?
3. Bạn sắp phải mô tả hạ tầng bằng YAML thay vì gõ lệnh — cú pháp đó khác gì so với bash script bạn quen viết?

## Phạm vi ôn tập (Scope)

Để tránh lặp lại nội dung đã học, module này chỉ tập trung vào phần liên quan trực tiếp tới automation:

| Chủ đề | Đã học sâu ở đâu (Linux Sysadmin) | Module này bổ sung gì |
|---|---|---|
| SSH cơ bản, SSH key | Module về SSH & Remote Access | Góc nhìn: SSH key cho service account automation, không phải user cá nhân |
| SSH Hardening | Module Security Hardening | Cách cấu hình ~/.ssh/config để quản lý nhiều host cùng lúc có tổ chức |
| Sudo, /etc/sudoers | Module User & Group Management | Mô hình sudo riêng cho automation (NOPASSWD có kiểm soát, giới hạn lệnh) |
| YAML | Chưa học (mới) | Toàn bộ cú pháp cơ bản — nền tảng bắt buộc cho mọi module còn lại |
| Jinja2 | Chưa học (mới) | Cú pháp cơ bản — học sâu hơn ở Module 04 |
| Inventory Planning | Chưa học (mới, nhưng dùng lại tư duy phân nhóm host) | Cách tư duy nhóm host theo môi trường/vai trò trước khi viết Ansible inventory thật |

## Mục tiêu học tập

Sau module này, bạn sẵn sàng bước vào Module 02 với: một SSH key riêng cho automation đã hoạt động, một file ~/.ssh/config gọn gàng, hiểu rõ mô hình sudo sẽ dùng cho Ansible, đọc-viết được YAML cơ bản không cần tra cứu, và không bỡ ngỡ khi thấy cú pháp {{ bien }} lần đầu tiên.
