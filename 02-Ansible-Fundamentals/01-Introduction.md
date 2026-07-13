---
title: "01 - Introduction"
module: 2
tags: [ansible, sysops-infra, module-02, introduction]
---

# 01 — Giới thiệu

## Bài toán doanh nghiệp thật: "300 server, 3 kỹ sư"

Hình dung bạn là sysadmin duy nhất phụ trách 300 server production. Có một lỗ hổng bảo mật OpenSSL cần vá **hôm nay**, trên toàn bộ 300 máy, không được sai sót, không được bỏ sót máy nào, và phải ghi lại máy nào đã vá xong để báo cáo audit. SSH thủ công vào từng máy là bất khả thi trong khung thời gian yêu cầu. Đây chính xác là bài toán Ansible sinh ra để giải quyết — và là lý do bạn học module này.

## Ansible là gì (đúng nghĩa, không marketing)

Ansible là công cụ automation mã nguồn mở, dùng để: cấu hình hệ thống (Configuration Management), triển khai ứng dụng (Application Deployment), và điều phối tác vụ đa hệ thống (Orchestration). Điểm khác biệt cốt lõi so với việc viết bash script tự chế: Ansible có mô hình **idempotent** built-in (chạy lại nhiều lần cho cùng kết quả, không gây side-effect), có ngôn ngữ mô tả trạng thái mong muốn (declarative, học ở Module 00), và có thư viện hàng nghìn module đã được kiểm thử cho hầu hết tác vụ hệ thống phổ biến.

## Vì sao Ansible, không phải Puppet/Chef/SaltStack?

| Tiêu chí | Ansible | Puppet / Chef | SaltStack |
|---|---|---|---|
| Kiến trúc | Agentless (SSH) | Agent-based (daemon chạy thường trực) | Agent-based (thường), có agentless mode |
| Ngôn ngữ | YAML | Ruby DSL (Puppet), Ruby (Chef) | YAML + Python |
| Độ dốc học tập | Thấp — YAML dễ đọc | Cao hơn — cần hiểu Ruby DSL | Trung bình |
| Cần cài gì trên managed node | Chỉ cần Python + SSH server (thường có sẵn) | Cần cài agent riêng, đăng ký với master | Cần cài minion (agent) |
| Phổ biến trong khảo sát DevOps 2025-2026 | Cao, đặc biệt ở SMB và team hạ tầng hỗn hợp | Ổn định ở doanh nghiệp lớn dùng lâu năm | Giảm dần, cộng đồng thu hẹp |

Ansible không phải "tốt nhất tuyệt đối" — nó đánh đổi hiệu năng ở quy mô rất lớn (do SSH overhead) để lấy độ đơn giản triển khai (không cần cài agent trước). Với đa số tổ chức vừa và nhỏ, và với vai trò cầu nối sang Kubernetes (khóa tiếp theo), Ansible là lựa chọn thực tế nhất để học sâu.

## Ansible trong bức tranh khóa 04

Module 00 đã trả lời "tại sao IaC". Module 01 đã chuẩn bị "SSH/YAML sẵn sàng chưa". Module 02 này trả lời câu hỏi "Ansible thật sự vận hành như thế nào" — từ đây trở đi, mọi khái niệm (Playbook, Template, Vault, Role) đều là lớp xây thêm trên 4 viên gạch nền: **Control Node, Managed Node, Inventory, Module**.

## Mục tiêu học tập

Sau module này, bạn cài đặt được Ansible, hiểu rõ 4 khái niệm nền tảng, viết được inventory tĩnh cho một nhóm server thật, phân biệt Variables với Facts, và chạy thành thạo ad-hoc command cho các tác vụ kiểm tra/sửa lỗi nhanh — nền tảng bắt buộc trước khi viết Playbook đầu tiên ở Module 03.
