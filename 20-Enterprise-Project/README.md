---
title: "Module 20 - Enterprise Project"
tags: [docker, ansible, sysops-infra, module-20, capstone, enterprise-project]
module: 20
part: "III - Operations"
difficulty: Advanced
status: draft
created: 2026-07-12
prerequisites: ["[[../19-Troubleshooting/README|Module 19]]"]
---

# Module 20 — Enterprise Project (Đồ án cuối khóa)

> [!info] Vị trí trong giáo trình
> Part III — Operations · Module 20/21 (module cuối cùng) của khóa 04 (System Operations & Infrastructure) · Đây là **bài kiểm tra tổng hợp** toàn bộ khóa học — không giới thiệu công cụ mới, mà yêu cầu bạn tự vận dụng mọi kiến thức đã học từ Module 00 tới Module 19 để tự tay dựng một hạ tầng doanh nghiệp hoàn chỉnh.

> [!warning] Đây là đồ án, không phải bài học lý thuyết
> File [[05-Labs]] của module này là nội dung dự án — chỉ chứa mô tả kiến trúc, yêu cầu kỹ thuật, các bước triển khai và tiêu chí hoàn thành, **không có câu hỏi lý thuyết nào**. File [[06-Troubleshooting]] gồm 20 tình huống sự cố thực tế được viết dưới dạng **kịch bản** (mô tả triệu chứng, yêu cầu bạn tự chẩn đoán), không phải dạng hỏi-đáp. Câu hỏi phỏng vấn tổng hợp toàn khóa vẫn được giữ riêng ở [[07-Interview]] như mọi module khác.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Bối cảnh dự án, vai trò của bạn, mục tiêu học tập |
| [[02-Theory]] | Ôn tập lý thuyết nền cho từng thành phần của kiến trúc 5 VM |
| [[03-Architecture]] | Sơ đồ kiến trúc tổng thể 5 VM — sơ đồ quan trọng nhất toàn khóa |
| [[04-Commands]] | Bộ lệnh tham khảo nhanh cho từng giai đoạn triển khai |
| [[05-Labs]] | **Nội dung dự án** — kiến trúc, yêu cầu, các bước triển khai, tiêu chí hoàn thành |
| [[06-Troubleshooting]] | 20 tình huống sự cố thực tế (dạng kịch bản, tự chẩn đoán) |
| [[07-Interview]] | Câu hỏi phỏng vấn tổng hợp toàn khóa |
| [[08-Summary]] | Tổng kết toàn khóa 21 module, hướng đi tiếp theo |

## Learning Objectives (tổng)

Sau khi hoàn thành module này, người học phải:

1. Tự dựng được 5 VM (Control Node, HAProxy, Web01, Web02, Database) và cấu hình mạng nội bộ giữa chúng.
2. Tự viết được bộ Ansible hoàn chỉnh (SSH key, Inventory, Roles, Playbooks, Templates) để triển khai tự động toàn bộ 4 node còn lại từ Control Node.
3. Tự deploy được các dịch vụ Docker (Nginx, MySQL, Redis, RabbitMQ) đúng vai trò từng node, dùng Docker Compose cho kiến trúc multi-container (network, volume).
4. Tự gắn được giám sát (Prometheus, Grafana) và backup (rsync, volume backup) cho toàn bộ hạ tầng vừa dựng.
5. Tự chẩn đoán được 20 tình huống sự cố thực tế trên chính hạ tầng mình dựng, áp dụng đúng quy trình debug phân tầng đã học ở Module 19.
6. Tổng kết lại được toàn bộ hành trình 21 module, biết chính xác mình đang đứng ở đâu trong lộ trình sự nghiệp Sysadmin/DevOps.

## Self-Review

> [!note] Self-Review
> - **Technical Accuracy:** Kiến trúc 5 VM, bộ dịch vụ Docker (Nginx, MySQL, Redis, RabbitMQ), và quy trình Ansible → Docker → Monitoring → Backup được thiết kế nhất quán với toàn bộ kiến thức đã xác nhận qua WebSearch ở các Module 16-19 trước đó (Harbor/registry, Grafana Alloy/Loki/Prometheus/cAdvisor, Rootless/Seccomp/AppArmor/Docker Bench). Không giới thiệu công cụ mới chưa được kiểm chứng ở module nào trước đó.
> - **Completeness:** Đã bao phủ đầy đủ yêu cầu: dựng 5 VM, triển khai bằng Ansible (SSH Key, Inventory, Roles, Playbooks, Templates), deploy Docker (Nginx, MySQL, Redis, RabbitMQ), Docker Compose (multi-container, network, volume), Monitoring (Prometheus, Grafana), Backup (rsync, volume backup), và 20 tình huống troubleshooting thực tế viết dạng kịch bản. [[05-Labs]] xác nhận không chứa câu hỏi lý thuyết/quiz nào.
> - **ASCII Diagram:** Sơ đồ kiến trúc tổng thể 5 VM trong [[03-Architecture]] đã được dựng bằng script Python (hàm `box()` đảm bảo `len()` border trên = border dưới = mọi dòng nội dung cho từng khung) và chạy `verify_boxes.py` xác nhận "ALL BOXES OK" — đây là sơ đồ quan trọng nhất toàn khóa nên được kiểm tra kỹ nhất.

## Cấu trúc thư mục

```
20-Enterprise-Project/
├── README.md
├── 01-Introduction.md
├── 02-Theory.md
├── 03-Architecture.md
├── 04-Commands.md
├── 05-Labs.md
├── 06-Troubleshooting.md
├── 07-Interview.md
├── 08-Summary.md
├── images/
├── labs/
└── scripts/
```
