---
title: "01 - Introduction"
module: 7
tags: [ansible, sysops-infra, module-07, introduction]
---

# 01 — Giới thiệu

## Khoảng cách giữa Lab và Production

Suốt Module 01-06, bạn triển khai lên 2-3 server lab, chấp nhận vài giây gián đoạn khi restart service — không ai bị ảnh hưởng. Trong production doanh nghiệp thật, một website thương mại điện tử có 20 web server phục vụ hàng nghìn người dùng cùng lúc: nếu bạn chạy Playbook restart `nginx` trên cả 20 server **cùng một lúc**, toàn bộ website sập trong vài giây đó — với site có doanh thu, vài giây này có thể tương đương thiệt hại tài chính thực tế và mất niềm tin khách hàng. Đây là bài toán trung tâm của module cuối cùng: **triển khai thay đổi trên hạ tầng đang phục vụ người dùng, mà không gây gián đoạn (Zero Downtime)**.

## Ba khái niệm cốt lõi của module này

- **Multi-tier Architecture:** hạ tầng doanh nghiệp thật hiếm khi chỉ có "web server" — thường có Load Balancer (phân phối traffic), Web/App tier (xử lý logic), Database tier (lưu trữ dữ liệu), mỗi tầng có đặc thù triển khai riêng.
- **Rolling Update:** cập nhật **từng phần nhỏ** hạ tầng tuần tự, không phải toàn bộ cùng lúc — công cụ chính là `serial` trong Playbook.
- **Zero Downtime:** kết hợp Rolling Update với Load Balancer — gỡ một node khỏi pool trước khi cập nhật, kiểm tra sức khỏe (health check) trước khi đưa trở lại, đảm bảo tại mọi thời điểm luôn có đủ node đang phục vụ traffic.

## Đây là module tổng hợp, không phải module khái niệm mới

Gần như mọi công cụ dùng ở module này đã học ở các module trước: Role (Module 06) để đóng gói từng tầng, Template (Module 04) để cấu hình HAProxy/Nginx động, Block/Rescue (Module 03) để xử lý lỗi rollback khi một node cập nhật thất bại, Vault (Module 05) để bảo vệ mật khẩu database. Điểm mới duy nhất thực sự: **tư duy vận hành ở quy mô doanh nghiệp** — hiểu rủi ro thật, không chỉ hiểu cú pháp.

## Mục tiêu học tập

Sau module này, bạn thiết kế và triển khai được một hệ thống 3 tầng hoàn chỉnh bằng Ansible, thực hiện Rolling Update có kiểm chứng zero downtime qua Load Balancer, và tích hợp Backup/Monitoring như một phần tự nhiên của quy trình triển khai — khép lại Part I với năng lực triển khai một hệ thống production thực tế, sẵn sàng bước sang Part II (Docker, Module 08).
