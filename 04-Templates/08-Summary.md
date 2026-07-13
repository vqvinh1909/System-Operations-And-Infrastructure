---
title: "08 - Summary"
module: 4
tags: [ansible, sysops-infra, module-04, summary]
---

# 08 — Tóm tắt Module 04

## Những gì đã học

- **Jinja2 nâng cao:** `{% if %}`/`{% for %}` trong file `.j2` thật, luôn phải đóng tag tường minh.
- **Module `template`:** render Jinja2 trước khi ghi, khác `copy` (chép nguyên văn) — nên dùng `owner`/`group`/`mode`/`validate` tường minh.
- **Filters:** biến đổi giá trị ngay trong biểu thức (`default`, `join`, `upper`, `to_json`, `regex_replace`...).
- **Lookup:** lấy dữ liệu từ Control Node (file, biến môi trường, output lệnh) — không phải Managed Node.
- **4 module xử lý file:** `file` (thuộc tính), `copy` (chép tĩnh), `template` (render động), `fetch` (kéo ngược từ Managed Node), `synchronize` (đồng bộ rsync).

## Bảng đối chiếu nhanh 4 module xử lý file

| Module | Chiều dữ liệu | Xử lý Jinja2 | Dùng khi |
|---|---|---|---|
| `file` | Không di chuyển | Không | Quản lý quyền, tạo thư mục/symlink |
| `copy` | Control → Managed | Không | File tĩnh, giống nhau mọi host |
| `template` | Control → Managed | Có | File cần khác nhau theo host/biến |
| `fetch` | Managed → Control | Không | Thu thập log/artifact tập trung |
| `synchronize` | Cả hai chiều | Không | Thư mục lớn, cần hiệu năng cao |

## Điều kiện để chuyển sang Module 05

1. Viết được template `.j2` với `{% if %}`/`{% for %}` chạy đúng, không lỗi cú pháp?
2. Dùng thành thạo ít nhất 5 filter khác nhau, biết khi nào cần `default()`?
3. Phân biệt rõ Lookup (Control Node) và Facts (Managed Node)?
4. Chọn đúng module trong 5 module xử lý file cho một tình huống cụ thể mà không cần tra cứu?

## Cầu nối sang Module 05

Ở Lab 4 module này, bạn đã thấy vấn đề: nếu cần lưu một giá trị bí mật cố định (session secret, mật khẩu database) để dùng trong template, bạn **không thể** để nó ở dạng plaintext trong file biến commit lên Git. Module 05 — Variables, Vault & Secrets giải quyết chính xác vấn đề này: thứ tự ưu tiên biến đầy đủ (variable precedence), và Ansible Vault — công cụ mã hóa file/biến ngay trong hệ sinh thái Ansible, cho phép commit secret đã mã hóa vào Git một cách an toàn.

## Điều hướng

- Quay lại: [[README|Module 04 — README]]
- Tiếp theo: [[../05-Variables-Vault-Secrets/README|Module 05 — Variables, Vault & Secrets]]
