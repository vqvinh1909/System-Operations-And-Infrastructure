---
title: "Module 19 - Troubleshooting"
tags: [docker, sysops-infra, module-19, troubleshooting, debugging, performance]
module: 19
part: "III - Operations"
difficulty: Advanced
status: draft
created: 2026-07-12
prerequisites: ["[[../18-Security/README|Module 18]]"]
next: "[[../20-Enterprise-Project/README|Module 20]]"
---

# Module 19 — Troubleshooting

> [!info] Vị trí trong giáo trình
> Part III — Operations · Module 19/21 của khóa 04 (System Operations & Infrastructure) · Nối tiếp Module 18 (Security) — bạn đã biết quan sát và bảo vệ container, module này tổng hợp và đào sâu **kỹ năng chẩn đoán sự cố** khi mọi lớp phòng thủ và giám sát vẫn không ngăn được vấn đề xảy ra.

> [!warning] Khác với các mục "Troubleshooting" nhỏ ở những module trước
> Từ Module 08 đến Module 18, mỗi module có một file `06-Troubleshooting.md` riêng, chỉ xử lý sự cố **trong phạm vi chủ đề module đó** (ví dụ Module 16 chỉ nói lỗi push/pull registry). Module 19 **tổng hợp lại** toàn bộ kỹ năng debug rải rác đó, đồng thời đào sâu hơn nhiều — dạy bạn **quy trình chẩn đoán có hệ thống** áp dụng được cho bất kỳ sự cố Docker nào bạn gặp trong công việc thật, không giới hạn theo từng công cụ cụ thể. File [[06-Troubleshooting]] của module này vì vậy dài và chi tiết hơn hẳn các module khác — đây là nội dung chính, không phải phụ lục.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Vì sao cần một quy trình debug có hệ thống, mục tiêu học tập |
| [[02-Theory]] | Mô hình 4 tầng debug: Container, Network, Storage, Performance |
| [[03-Architecture]] | Sơ đồ mô hình phân tầng, luồng chẩn đoán network/performance |
| [[04-Commands]] | Bộ lệnh chẩn đoán đầy đủ cho từng tầng |
| [[05-Labs]] | Lab tự gây lỗi có kiểm soát và tự chẩn đoán ở cả 4 tầng |
| [[06-Troubleshooting]] | **Nội dung chính** — Container/Network/Storage/Performance Debugging chuyên sâu |
| [[07-Interview]] | Câu hỏi phỏng vấn về debug Docker |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 20 (Enterprise Project) |

## Learning Objectives (tổng)

Sau khi hoàn thành module này, người học phải:

1. Áp dụng được mô hình debug phân tầng (Application → Container → Network → Storage → Host/Kernel) để khoanh vùng nguyên nhân sự cố có hệ thống, thay vì đoán mò.
2. Chẩn đoán được sự cố container không khởi động được, restart loop, exit code bất thường bằng `docker inspect`, `docker logs`, `docker events`.
3. Chẩn đoán được sự cố mạng giữa các container: DNS nội bộ, network không chung, iptables/NAT sai, port mapping xung đột.
4. Chẩn đoán được sự cố storage: volume permission sai, disk đầy, overlay2 lỗi, volume mồ côi (orphaned).
5. Phân tích được hiệu năng container: đọc `docker stats`, đọc trực tiếp cgroup, phân biệt bottleneck do CPU, RAM, hay I/O, phân biệt do giới hạn cấu hình sai hay do host thật sự quá tải.
6. Tổng hợp toàn bộ quy trình trên thành một checklist debug áp dụng được ngay trong công việc thực tế, chuẩn bị trực tiếp cho Module 20 (dự án cuối khóa có 20 tình huống troubleshooting thực tế).

## Self-Review

> [!note] Self-Review
> - **Technical Accuracy:** Nội dung dựa trên hành vi thật của Docker Engine, cgroup v1/v2, iptables/nftables mà Docker daemon tự quản lý — các lệnh chẩn đoán (`docker inspect`, `docker stats`, `/sys/fs/cgroup/...`) đã đối chiếu với tài liệu chính thức Docker Engine và kiến thức cgroup đã học ở Part II (Container Technology). Không bịa số liệu benchmark nào không kiểm chứng được.
> - **Completeness:** Đã bao phủ đầy đủ 4 tầng debug được yêu cầu: Container Debugging, Network Debugging, Storage Debugging, Performance Analysis, với độ sâu và độ rộng lớn hơn hẳn các file `06-Troubleshooting.md` nhỏ ở từng module trước, đúng như phạm vi đề ra.
> - **ASCII Diagram:** Mọi sơ đồ trong [[03-Architecture]] đã được dựng bằng script Python (hàm `box()` đảm bảo `len()` border trên = border dưới = mọi dòng nội dung cho từng khung) và chạy script `verify_boxes.py` xác nhận "ALL BOXES OK" trước khi đưa vào file.

## Cấu trúc thư mục

```
19-Troubleshooting/
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
