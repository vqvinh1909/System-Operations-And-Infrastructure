---
title: "08 - Summary"
module: 5
tags: [ansible, sysops-infra, module-05, summary]
---

# 08 — Tóm tắt Module 05

## Những gì đã học

- **Variable Precedence:** 22 cấp, từ `role defaults` (thấp nhất) tới `extra-vars` (cao nhất) — hai nguyên tắc thực dụng: extra-vars luôn thắng, role defaults luôn thua.
- **Ansible Vault:** mã hóa AES256, mã hóa toàn bộ file (`create`/`encrypt`/`edit`/`view`/`decrypt`) hoặc biến đơn lẻ (`encrypt_string`).
- **Multiple Vault ID:** quản lý nhiều password cho nhiều môi trường bằng cú pháp `ID@SOURCE`.
- **Rotate:** `ansible-vault rekey` để xoay vòng password mã hóa định kỳ.
- **Secrets Management:** không commit plaintext, tách biệt theo môi trường, giới hạn người biết password, xử lý khẩn cấp khi lộ secret.

## Bảng đối chiếu 2 cách dùng Vault

| Cách dùng | Khi nào chọn | Đánh đổi |
|---|---|---|
| `encrypt` cả file | Phần lớn biến trong file đều nhạy cảm | Không review được diff trong Pull Request |
| `encrypt_string` từng biến | Chỉ 1-2 biến nhạy cảm trong file lớn | Tốn công xác định đúng biến cần mã hóa |

## Điều kiện để chuyển sang Module 06

1. Giải thích được kết quả precedence cho một tình huống biến khai báo ở 3+ nơi mà không cần tra bảng?
2. Tạo, sửa, dùng thành thạo file Vault trong Playbook?
3. Biết chọn giữa `encrypt` cả file và `encrypt_string` cho từng tình huống?
4. Quản lý được nhiều Vault ID cho nhiều môi trường?
5. Trình bày được quy trình xử lý khi phát hiện secret bị commit nhầm plaintext?

## Cầu nối sang Module 06

Tới thời điểm này, mọi Playbook bạn viết đều nằm trong **một dự án đơn lẻ**. Nếu công ty có 10 dự án khác nhau đều cần "cài nginx đúng chuẩn công ty", bạn sẽ copy-paste cùng một đoạn Task 10 lần — vi phạm chính nguyên tắc "single source of truth" đã học từ Module 00. Module 06 — Roles dạy cách **đóng gói** Task, Handler, Template, Vars (bao gồm cả Vault) thành một đơn vị tái sử dụng chuẩn hóa, chia sẻ được qua Ansible Galaxy — bước cuối cùng trước khi Part I khép lại bằng Module 07 (Enterprise Automation), nơi mọi kỹ năng từ Module 01 tới 06 được ráp lại thành một hệ thống triển khai doanh nghiệp hoàn chỉnh.

## Điều hướng

- Quay lại: [[README|Module 05 — README]]
- Tiếp theo: [[../06-Roles/README|Module 06 — Roles]]
