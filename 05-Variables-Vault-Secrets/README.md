---
title: "Module 05 - Variables, Vault & Secrets"
tags: [ansible, sysops-infra, module-05, variables, vault, secrets, precedence]
module: 5
part: "I - Infrastructure as Code (Ansible)"
difficulty: Intermediate
status: draft
created: 2026-07-13
prerequisites: ["[[../04-Templates/README|Module 04]]"]
next: "[[../06-Roles/README|Module 06]]"
---

# Module 05 — Variables, Vault & Secrets

> [!info] Vị trí trong giáo trình
> Module 05/07 của Part I. Các module trước dùng biến khá "hồn nhiên" — khai báo ở đâu cũng được, miễn chạy đúng. Module này trả lời hai câu hỏi bắt buộc phải rõ ràng trước khi đưa Ansible vào production: (1) khi một biến được khai báo ở nhiều nơi, **giá trị nào thắng**? (2) làm sao lưu mật khẩu/API key trong Git **mà không lộ plaintext**? Đây là module về kỷ luật vận hành, không chỉ là cú pháp.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Vì sao "biến ở đâu cũng chạy được" là rủi ro, không phải sự linh hoạt |
| [[02-Theory]] | Variable Precedence đầy đủ 22 cấp, Ansible Vault (mã hóa file/biến/string), Secrets Management best practices |
| [[03-Architecture]] | Sơ đồ thứ tự ưu tiên biến, sơ đồ luồng mã hóa/giải mã Vault |
| [[04-Commands]] | `ansible-vault create/edit/encrypt/decrypt/rekey/view`, `--vault-id`, `ansible-config` kiểm tra precedence |
| [[05-Labs]] | Tạo Vault file, mã hóa biến đơn lẻ, tích hợp Vault vào Playbook, thử nghiệm precedence thực tế |
| [[06-Troubleshooting]] | Lỗi sai password Vault, lỗi commit nhầm plaintext, lỗi precedence gây bug khó hiểu |
| [[07-Interview]] | Câu hỏi phỏng vấn Variable Precedence, Vault, Secrets Management |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 06 |

## Learning Objectives (tổng)

1. Liệt kê và giải thích đúng thứ tự ưu tiên biến (Variable Precedence) của Ansible, từ thấp nhất (role defaults) tới cao nhất (extra-vars dòng lệnh).
2. Debug được một tình huống "biến ra giá trị sai không hiểu vì sao" bằng cách truy vết đúng theo bảng precedence.
3. Sử dụng `ansible-vault` để mã hóa toàn bộ file biến, hoặc chỉ mã hóa một biến đơn lẻ trong file không mã hóa (`encrypt_string`).
4. Tích hợp file đã mã hóa Vault vào Playbook một cách trong suốt (Ansible tự giải mã lúc chạy, không cần thay đổi cú pháp Task).
5. Áp dụng `--vault-id` để quản lý nhiều password Vault khác nhau cho nhiều môi trường (dev/staging/production).
6. Trình bày được các nguyên tắc Secrets Management best practices: không commit plaintext, rotate định kỳ, tách biệt secret theo môi trường, hạn chế người biết password Vault.

## Self-Review

> [!note] Self-Review
> - **Accuracy:** Đã WebSearch xác nhận qua Ansible Community Documentation (07/2026): Ansible Vault dùng cipher **AES256** (thuật toán mã hóa duy nhất được hỗ trợ chính thức), cú pháp `--vault-id ID@SOURCE`, biến cấu hình `ANSIBLE_VAULT_PASSWORD_FILE`, lệnh `ansible-vault rekey` để xoay vòng password mã hóa. Không có thay đổi thuật toán mã hóa hay cấu trúc lệnh CLI cốt lõi so với các bản ansible-core gần đây.
> - **Diagram:** Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã đo bằng script Python trước khi đưa vào.
> - **Cấm:** [[05-Labs]] chỉ chứa nội dung kỹ thuật, không có câu hỏi phỏng vấn.

## Điều hướng

- Quay lại: [[../04-Templates/README|Module 04 — Templates]]
- Tiếp theo: [[../06-Roles/README|Module 06 — Roles]]
