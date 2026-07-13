---
title: "Module 03 - Playbooks"
tags: [ansible, sysops-infra, module-03, playbooks, tasks, handlers, loops, conditionals]
module: 3
part: "I - Infrastructure as Code (Ansible)"
difficulty: Intermediate
status: draft
created: 2026-07-13
prerequisites: ["[[../02-Ansible-Fundamentals/README|Module 02]]"]
next: "[[../04-Templates/README|Module 04]]"
---

# Module 03 — Playbooks

> [!info] Vị trí trong giáo trình
> Module 03/07 của Part I. Ở Module 02, bạn gọi từng module riêng lẻ bằng ad-hoc command. Module này dạy cách **gộp nhiều task thành một quy trình có thể lặp lại, chia sẻ, đưa vào version control** — chính là Playbook, đơn vị làm việc trung tâm của mọi kỹ sư Ansible trong thực tế. Đây là module dài và quan trọng nhất Part I — Templates, Vault, Roles ở các module sau đều là các khối xây thêm bên trong hoặc xung quanh Playbook.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Vì sao cần Playbook thay vì chuỗi ad-hoc command |
| [[02-Theory]] | Cấu trúc Playbook, Task, Variables trong Playbook, Loops, Conditionals, Blocks, Handlers, Tags, Include/Import, Error Handling |
| [[03-Architecture]] | Sơ đồ vòng đời thực thi Playbook, luồng Handler, luồng Block/Rescue/Always |
| [[04-Commands]] | `ansible-playbook`, cờ thường dùng, `--check`, `--diff`, `--tags`, `--limit` |
| [[05-Labs]] | Viết Playbook cài web server, dùng loop/conditional/handler/block thực tế |
| [[06-Troubleshooting]] | Lỗi cú pháp Task, lỗi Handler không chạy, lỗi thứ tự Include/Import |
| [[07-Interview]] | Câu hỏi phỏng vấn Playbook, Handler, Block/Rescue |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 04 |

## Learning Objectives (tổng)

1. Viết được cấu trúc Playbook hợp lệ: `hosts`, `become`, `vars`, `tasks` — hiểu Play là gì, Playbook có thể chứa nhiều Play.
2. Sử dụng Loops (`loop`) để lặp một Task trên danh sách giá trị, thay vì viết lặp lại nhiều Task giống nhau.
3. Sử dụng Conditionals (`when`) để chạy Task có điều kiện dựa trên Variables hoặc Facts.
4. Dùng Handlers để chỉ thực thi hành động (ví dụ restart service) khi có Task thực sự gây thay đổi (`notify`/`changed`).
5. Tổ chức Task bằng Blocks, và xử lý lỗi có cấu trúc bằng `rescue`/`always` — mô hình try/except/finally của Ansible.
6. Phân biệt `ignore_errors` và `failed_when`, biết khi nào dùng cái nào.
7. Phân biệt Include (động, resolve lúc chạy) và Import (tĩnh, resolve lúc parse) cho Task và Playbook, hiểu tác động tới Tags và luồng chạy.
8. Dùng Tags để chạy chọn lọc một phần Playbook mà không cần chạy toàn bộ.

## Self-Review

> [!note] Self-Review
> - **Accuracy:** Đã WebSearch đối chiếu Ansible Community Documentation (bản 07/2026) về hành vi Handler (chạy ở cuối Play theo mặc định, hoặc ngay sau Task với `meta: flush_handlers`), cú pháp `block/rescue/always`, và sự khác biệt Include/Import — không có thay đổi hành vi cốt lõi so với các bản ansible-core gần đây (2.19-2.21).
> - **Diagram:** Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã đo bằng script Python trước khi đưa vào.
> - **Cấm:** [[05-Labs]] chỉ chứa nội dung kỹ thuật, không có câu hỏi phỏng vấn — toàn bộ câu hỏi tập trung ở [[07-Interview]].

## Điều hướng

- Quay lại: [[../02-Ansible-Fundamentals/README|Module 02 — Ansible Fundamentals]]
- Tiếp theo: [[../04-Templates/README|Module 04 — Templates]]
