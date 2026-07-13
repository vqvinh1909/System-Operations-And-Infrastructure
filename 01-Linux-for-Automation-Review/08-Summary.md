---
title: "08 - Summary"
module: 1
tags: [ansible, sysops-infra, module-01, summary]
---

# 08 — Tóm tắt Module 01

## Những gì đã học

- **SSH cho automation:** key-based authentication là bắt buộc, cần cặp key riêng tách biệt khỏi key cá nhân, `~/.ssh/config` giúp quản lý nhiều host có tổ chức, `ssh-keyscan` tránh treo automation do `known_hosts`.
- **Sudo cho service account:** thiết kế user chuyên dụng (`deploy`/`ansible_svc`), NOPASSWD có kiểm soát, đặt nền cho `become: true` sẽ dùng xuyên suốt các module Playbook.
- **Inventory Planning:** tư duy phân nhóm host theo môi trường / vai trò / vị trí — một host thuộc nhiều nhóm cùng lúc — chuẩn bị trực tiếp cho Inventory thật ở Module 02.
- **YAML:** scalar, list, dictionary, thụt lề bằng space, lỗi thường gặp (tab/space lẫn lộn, thụt lề lệch âm thầm đổi cấu trúc).
- **Jinja2 cơ bản:** `{{ }}` xuất giá trị, `{% %}` điều khiển luồng, `{# #}` comment — lý do phải đặt trong ngoặc kép khi dùng trong YAML.

## Bảng đối chiếu nhanh

| Khái niệm | Bạn đã biết (Linux Sysadmin) | Bạn vừa học thêm (Module 01) |
|---|---|---|
| SSH | Đăng nhập, hardening | Key riêng cho automation, `known_hosts` không tương tác |
| Sudo | Cấu hình cho user thật | Service account, NOPASSWD có kiểm soát |
| Phân nhóm host | — | Tư duy multi-axis (env/role/location) trước khi viết inventory |
| YAML | — (mới) | Cú pháp nền tảng của toàn bộ Ansible |
| Jinja2 | — (mới) | Nhận diện cú pháp cơ bản, học sâu ở Module 04 |

## Điều kiện để chuyển sang Module 02

Bạn nên tự tin trả lời "có" cho cả 5 câu sau trước khi qua Module 02:

1. SSH key automation đã hoạt động, kết nối không cần password tới ít nhất 2 máy?
2. `sudo -n whoami` trả về `root` trên các máy đó qua service account?
3. Viết được YAML dictionary lồng list mà không cần tra cứu?
4. Phân biệt được `{{ }}` và `{% %}` khi nhìn thấy trong một file bất kỳ?
5. Có thể mô tả bằng lời cách bạn sẽ nhóm 10-20 server thật theo môi trường/vai trò?

## Cầu nối sang Module 02

Module 02 — Ansible Fundamentals sẽ dùng lại **nguyên vẹn** hạ tầng SSH/sudo bạn vừa chuẩn bị: file `ansible.cfg` sẽ trỏ tới đúng SSH key này, inventory Ansible thật sẽ hiện thực hóa đúng mô hình phân nhóm bạn vừa luyện tập, và `become: true` trong module đầu tiên sẽ chạy trên đúng service account này.

## Điều hướng

- Quay lại: [[README|Module 01 — README]]
- Tiếp theo: [[../02-Ansible-Fundamentals/README|Module 02 — Ansible Fundamentals]]
