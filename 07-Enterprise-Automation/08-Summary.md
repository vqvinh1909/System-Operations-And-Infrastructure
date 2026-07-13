---
title: "08 - Summary"
module: 7
tags: [ansible, sysops-infra, module-07, summary, part-1-recap]
---

# 08 — Tóm tắt Module 07 và toàn bộ Part I

## Những gì đã học ở Module 07

- **Multi-tier Architecture:** tách Load Balancer / Web-App / Database tier, mỗi tầng có chiến lược triển khai riêng.
- **Rolling Update:** `serial` (cố định, phần trăm, hoặc progressive/canary) + `max_fail_percentage` giới hạn phạm vi ảnh hưởng.
- **Zero Downtime:** chuỗi 6 bước — gỡ khỏi pool, drain, cập nhật, restart, health check (`until`/`retries`), đưa lại vào pool.
- **HAProxy/Nginx qua Template:** dùng `groups`/`hostvars` để một node "biết" về node khác.
- **Database Tier:** không rolling update tùy tiện, backup bắt buộc trước thay đổi, chiến lược tương thích ngược cho thay đổi schema.
- **Backup & Monitoring tích hợp:** backup có kiểm tra kết quả (`block/rescue`), thông báo triển khai qua `run_once` + `delegate_to`.

## Bảng tổng kết toàn bộ Part I — 7 module, một hệ thống hoàn chỉnh

| Module | Kỹ năng cốt lõi | Vai trò trong hệ thống cuối cùng (Module 07) |
|---|---|---|
| 01 — Linux for Automation Review | SSH key, sudo, YAML, Jinja2 cơ bản | Nền tảng kết nối và cú pháp cho mọi thứ phía sau |
| 02 — Ansible Fundamentals | Control/Managed Node, Inventory, Module, ansible.cfg | Inventory 3 tầng (loadbalancers/webservers/dbservers) |
| 03 — Playbooks | Task, Loop, Conditional, Block, Handler, Tags | Toàn bộ Task trong Playbook Rolling Update |
| 04 — Templates | Jinja2 nâng cao, Filter, Lookup, 4 module file | Render `haproxy.cfg.j2` động theo `groups`/`hostvars` |
| 05 — Variables, Vault & Secrets | Precedence, Ansible Vault | Bảo vệ mật khẩu database dùng trong backup/kết nối |
| 06 — Roles | Cấu trúc chuẩn, Dependencies, Galaxy | Đóng gói từng tầng thành Role tái sử dụng |
| 07 — Enterprise Automation | Multi-tier, Rolling Update, Zero Downtime | Ráp toàn bộ thành một pipeline triển khai production |

## Điều kiện hoàn thành Part I

1. Triển khai được kiến trúc 3 tầng hoàn chỉnh bằng Ansible, dùng lại Role từ Module 06?
2. Thực hiện Rolling Update có kiểm chứng thực tế bằng log, không chỉ tin vào lý thuyết?
3. Giải thích và triển khai được đầy đủ chuỗi Zero Downtime, xử lý đúng trường hợp lỗi?
4. Tích hợp Backup có kiểm tra kết quả, không để backup "báo thành công giả"?
5. Có thể tự tin nói: "Tôi có thể dùng Ansible triển khai một thay đổi lên hệ thống production đang phục vụ người dùng, mà không gây gián đoạn"?

## Cầu nối sang Part II — Docker

Part I đã trang bị toàn bộ kỹ năng Infrastructure as Code với Ansible — quản lý **máy chủ truyền thống** (VM, bare-metal) một cách tự động, nhất quán, an toàn. Part II (Module 08 trở đi, do agent khác xây dựng song song) chuyển sang **Docker** — đơn vị đóng gói ứng dụng khác biệt căn bản so với VM: container chia sẻ kernel host, khởi động nhanh hơn, và đặt ra bài toán triển khai/vận hành khác với những gì Part I vừa học. Nhiều khái niệm ở Part I vẫn áp dụng được (Ansible có thể dùng để triển khai và quản lý Docker host), nhưng Part II sẽ giới thiệu mô hình tư duy container-first — chuẩn bị trực tiếp cho khóa 05 (Kubernetes for SysOps) tiếp theo trong lộ trình 5 khóa.

## Điều hướng

- Quay lại: [[README|Module 07 — README]]
- Tiếp theo: [[../08-Container-Technology/README|Module 08 — Container Technology]]
