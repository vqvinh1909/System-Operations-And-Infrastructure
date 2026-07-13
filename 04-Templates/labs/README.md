---
title: "Labs - Working Directory"
module: 4
tags: [ansible, sysops-infra, module-04, labs]
---

# Labs — Thư mục làm việc

Đây là nơi lưu **code và config thực hành** khi làm các bài lab trong [[../05-Labs|05 - Labs]] của module "Templates (Jinja2)".

Gợi ý tổ chức:
- Mỗi bài lab tạo một thư mục con riêng (ví dụ `lab1/`, `lab2/`, `mini-project/`).
- Giữ lại inventory, playbook, template, output log của từng lab để đối chiếu khi ôn tập hoặc phỏng vấn.
- Không commit file chứa secret dạng plaintext (password, API key) lên Git — dùng Ansible Vault nếu cần lưu secret trong lab.
