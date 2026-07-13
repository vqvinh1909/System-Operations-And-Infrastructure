---
title: "Module 08 - Container Technology"
tags: [docker, sysops-infra, module-08, container, kernel]
module: 8
part: "II - Container Fundamentals"
difficulty: Intermediate
status: draft
created: 2026-07-12
prerequisites: ["[[../07-Enterprise-Automation/README|Module 07]]"]
next: "[[../09-Docker-Engine/README|Module 09]]"
---

# Module 08 - Container Technology

## Vì sao module này đứng đầu Part II

Trước khi gõ lệnh `docker run` đầu tiên, một Sysadmin nghiêm túc cần hiểu **container không phải công nghệ mới** — nó là cách đóng gói lại ba cơ chế đã có sẵn trong kernel Linux từ hơn một thập kỷ: **namespaces** (cô lập), **cgroups** (giới hạn tài nguyên), và **union filesystem** (chia sẻ layer). Docker chỉ là lớp tiện ích (tooling) làm cho ba thứ đó dễ dùng. Hiểu tận gốc giúp bạn:

- Debug được các lỗi "container không thấy network", "container ăn hết RAM node" mà không cần đoán mò.
- Trả lời phỏng vấn tự tin thay vì học thuộc lòng "container nhẹ hơn VM".
- Biết giới hạn thật của container (không phải VM, không cách ly kernel) để tư vấn đúng cho use case bảo mật cao.

## Nội dung

| # | File | Nội dung chính |
|---|------|-----------------|
| 1 | [[01-Introduction]] | Bài toán doanh nghiệp: vì sao container ra đời, "works on my machine" |
| 2 | [[02-Theory]] | VM vs Container, chuẩn OCI, Namespaces, Cgroups v1/v2, OverlayFS, Image Layer |
| 3 | [[03-Architecture]] | Sơ đồ kiến trúc, luồng syscall kernel khi tạo container |
| 4 | [[04-Commands]] | `unshare`, `nsenter`, `lsns`, đọc cgroup trong `/sys/fs/cgroup` |
| 5 | [[05-Labs]] | Lab tự dựng "container" bằng namespaces thủ công |
| 6 | [[06-Troubleshooting]] | Sự cố thật liên quan namespace/cgroup/overlay |
| 7 | [[07-Interview]] | Câu hỏi phỏng vấn Junior/Mid/Thực chiến |
| 8 | [[08-Summary]] | Tổng kết, checklist |
| - | [[labs/README|Labs folder]] | Hướng dẫn môi trường thực hành |
| - | [[scripts/README|Scripts folder]] | Script hỗ trợ |

## Mục tiêu học xong module

- Giải thích được sự khác biệt kiến trúc VM vs Container bằng sơ đồ, không học vẹt.
- Liệt kê và mô tả đúng chức năng 6 loại namespace Linux dùng cho container.
- Phân biệt cgroups v1 (hierarchy riêng từng subsystem) và cgroups v2 (unified hierarchy), biết driver `systemd` vs `cgroupfs`.
- Giải thích OverlayFS: lowerdir/upperdir/merged, vì sao image có nhiều layer chỉ-đọc (read-only).
- Tự tạo được một "container" tối giản bằng `unshare` để hiểu Docker thực chất đang gọi syscall gì.

## Điều kiện tiên quyết

- [[../07-Enterprise-Automation/README|Module 07]] — đã nắm Linux Sysadmin cơ bản, quản lý tiến trình, filesystem, quyền hạn.
- Có quyền `sudo` trên một máy Linux (VM Ubuntu 24.04 hoặc tương đương) để chạy lab namespace thủ công.

## Tiếp theo

Sau khi xong module này, sang [[../09-Docker-Engine/README|Module 09]] để tìm hiểu Docker Engine — công cụ dùng các cơ chế kernel này để đóng gói thành trải nghiệm `docker run` quen thuộc.

> [!note] Self-Review
> - Số liệu kỹ thuật (namespaces, cgroups v1/v2, OCI Spec, OverlayFS) đối chiếu với facts sheet đã xác nhận trước (không cần WebSearch lại): OCI Runtime Spec v1.3.0 (11/2025), OCI Image Spec v1.1.0 (2/2024), cgroups v2 mặc định trên distro hiện đại (systemd ≥ 247), driver khuyến nghị `systemd`.
> - Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã được đo bằng script Python (`len()` từng dòng khớp border trên/dưới) trước khi ghi file — xem chi tiết trong 03-Architecture.md.
