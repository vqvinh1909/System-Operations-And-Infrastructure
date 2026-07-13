---
title: "Module 00 - Infrastructure Automation Fundamentals"
tags: [ansible, sysops-infra, module-00, fundamentals, iac]
module: 0
part: "I - Infrastructure as Code (Ansible)"
difficulty: Foundation
status: draft
created: 2026-07-12
prerequisites: ["[[../../03-Linux System Administrator/28-Enterprise-Capstone-Project/README|Linux Sysadmin - Module 28]]"]
next: "[[../01-Linux-for-Automation-Review/README|Module 01]]"
---

# Module 00 — Infrastructure Automation Fundamentals

> [!info] Vị trí trong giáo trình
> Module đầu tiên của **khóa 04 — System Operations & Infrastructure**, cầu nối giữa khóa Linux System Administrator (đã xong) và Kubernetes. Đây là module **tư duy** — chưa gõ lệnh Ansible nào — mục tiêu là hiểu **tại sao** ngành chuyển từ quản trị máy chủ thủ công sang Infrastructure as Code (IaC), trước khi học **làm thế nào** ở Module 02 trở đi.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Giới thiệu, bối cảnh, mục tiêu học tập |
| [[02-Theory]] | Traditional Infra vs IaC, Lifecycle, Declarative vs Imperative, Agent vs Agentless, Idempotency, Immutable Infra, GitOps |
| [[03-Architecture]] | Sơ đồ so sánh workflow, vòng đời hạ tầng, kiến trúc agent vs agentless |
| [[04-Commands]] | Lệnh kiểm tra môi trường, khởi tạo repo IaC, demo idempotency bằng shell |
| [[05-Labs]] | Bài tập phân tích, demo idempotency, mini project thiết kế kiến trúc IaC |
| [[06-Troubleshooting]] | Configuration drift, snowflake server, lỗi tư duy thường gặp khi mới học IaC |
| [[07-Interview]] | Câu hỏi phỏng vấn khái niệm IaC/DevOps |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 01 |

## Learning Objectives (tổng)

1. Giải thích được vì sao doanh nghiệp chuyển từ quản trị hạ tầng thủ công ("ClickOps") sang Infrastructure as Code, gắn với chi phí thực tế (thời gian, lỗi con người, khả năng mở rộng).
2. Phân biệt Declarative và Imperative, biết khi nào Ansible dùng cách tiếp cận nào.
3. Phân biệt mô hình Agent-based và Agentless, giải thích vì sao Ansible chọn agentless.
4. Định nghĩa và giải thích Idempotency — nguyên lý cốt lõi xuyên suốt toàn bộ khóa học.
5. Phân biệt Mutable Infrastructure và Immutable Infrastructure.
6. Nắm khái niệm GitOps ở mức giới thiệu (single source of truth, Git làm nơi lưu trạng thái mong muốn).
7. Hình dung được kiến trúc IaC tổng thể trong một tổ chức (Enterprise IaC Architecture): Git repo, CI/CD, control node, target infrastructure.

## Self-Review

> [!note] Self-Review
> - **Accuracy:** Đã WebSearch xác nhận khái niệm precedence/agentless của Ansible qua tài liệu chính thức Ansible Community Documentation (cập nhật tháng 7/2026) để đảm bảo mô tả kiến trúc agentless nhất quán với các module Ansible sau này trong khóa. Các khái niệm IaC/GitOps/Immutable Infrastructure là kiến thức nền tảng ổn định, không phụ thuộc phiên bản phần mềm cụ thể nên không có rủi ro số liệu lỗi thời.
> - **Diagram:** 3 sơ đồ ASCII trong [[03-Architecture]] đã được đo bằng script Python (`len()` từng dòng trong khung) để đảm bảo border trên = border dưới = mọi dòng nội dung trước khi đưa vào file.

## Điều hướng

- Tiếp theo: [[../01-Linux-for-Automation-Review/README|Module 01 — Linux for Automation Review]]
