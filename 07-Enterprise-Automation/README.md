---
title: "Module 07 - Enterprise Automation"
tags: [ansible, sysops-infra, module-07, enterprise, rolling-update, haproxy, zero-downtime]
module: 7
part: "I - Infrastructure as Code (Ansible)"
difficulty: Advanced
status: draft
created: 2026-07-13
prerequisites: ["[[../06-Roles/README|Module 06]]"]
next: "[[../08-Container-Technology/README|Module 08]]"
---

# Module 07 — Enterprise Automation

> [!info] Vị trí trong giáo trình
> Module 07/07 — module cuối cùng, khép lại Part I (Infrastructure as Code với Ansible). Đây là module "ráp hệ thống": bạn dùng lại toàn bộ kỹ năng từ Module 01-06 (SSH, Inventory, Playbook, Template, Vault, Role) để triển khai một kiến trúc doanh nghiệp nhiều tầng thật sự — web tier phía sau load balancer, application tier, database tier — với Rolling Update không gây gián đoạn dịch vụ (Zero Downtime), tích hợp backup và giám sát. Sau module này, Part I kết thúc, khóa 04 chuyển sang Part II — Docker (Module 08 trở đi, agent khác xây dựng).

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Từ "Playbook chạy trên vài server lab" tới "triển khai production nhiều tầng an toàn" |
| [[02-Theory]] | Multi-tier Architecture, Rolling Update, Zero Downtime, HAProxy/Nginx làm Load Balancer, Database Deployment, Backup Strategy, Monitoring Deployment |
| [[03-Architecture]] | Sơ đồ kiến trúc 3 tầng, sơ đồ luồng Rolling Update qua load balancer, sơ đồ backup/monitoring |
| [[04-Commands]] | Cấu hình HAProxy qua Ansible, `serial`, `max_fail_percentage`, health check |
| [[05-Labs]] | Triển khai đầy đủ 3 tầng, Rolling Update có kiểm chứng zero downtime, backup tự động, giám sát cơ bản |
| [[06-Troubleshooting]] | Lỗi rolling update dừng giữa chừng, lỗi health check sai, lỗi backup không nhất quán |
| [[07-Interview]] | Câu hỏi phỏng vấn kiến trúc doanh nghiệp, Zero Downtime, chiến lược backup |
| [[08-Summary]] | Tổng kết toàn bộ Part I, cầu nối sang Module 08 |

## Learning Objectives (tổng)

1. Thiết kế Playbook triển khai kiến trúc Multi-tier (Load Balancer → Web/App tier → Database tier) bằng cách kết hợp nhiều Role đã học ở Module 06.
2. Áp dụng `serial` và `max_fail_percentage` để thực hiện Rolling Update — cập nhật từng phần nhỏ hạ tầng thay vì toàn bộ cùng lúc.
3. Thiết kế quy trình Zero Downtime Deployment: gỡ node khỏi load balancer trước khi cập nhật, health check trước khi đưa lại vào pool.
4. Cấu hình HAProxy/Nginx làm Load Balancer bằng Ansible Template (Module 04), hiểu cơ chế health check tầng ứng dụng.
5. Triển khai Database Tier có xét tới đặc thù khác biệt so với Web tier (không thể "rolling update" tùy tiện, cần chiến lược riêng cho backup/failover).
6. Tích hợp Backup tự động hóa và Monitoring Deployment cơ bản như một phần của Playbook triển khai, không phải bước thủ công tách rời.

## Self-Review

> [!note] Self-Review
> - **Accuracy:** Đã WebSearch xác nhận qua Ansible Community Documentation (07/2026) cú pháp `serial` (bao gồm dạng progressive như `serial: [1, 5, "25%"]`) và `max_fail_percentage` không đổi qua các bản ansible-core gần đây. Cấu hình HAProxy/Nginx health check dùng nguyên lý chuẩn ngành (không phụ thuộc phiên bản Ansible cụ thể).
> - **Diagram:** Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã đo bằng script Python trước khi đưa vào.
> - **Cấm:** [[05-Labs]] chỉ chứa nội dung kỹ thuật, không có câu hỏi phỏng vấn.

## Điều hướng

- Quay lại: [[../06-Roles/README|Module 06 — Roles]]
- Tiếp theo: [[../08-Container-Technology/README|Module 08 — Container Technology]]
