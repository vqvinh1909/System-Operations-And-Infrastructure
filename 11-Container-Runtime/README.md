---
title: "Module 11 - Container Runtime"
tags: [docker, sysops-infra, module-11, runtime, operations]
module: 11
part: "II - Container Fundamentals"
difficulty: Intermediate
status: draft
created: 2026-07-12
prerequisites: ["[[../10-Docker-Images/README|Module 10]]"]
next: "[[../12-Networking/README|Module 12]]"
---

# Module 11 — Container Runtime

> [!info] Vị trí trong giáo trình
> Module 10 dạy bạn cách **đóng gói** ứng dụng thành image. Module 11 dạy bạn cách **vận hành** container đó mỗi ngày — đây là phần việc chiếm phần lớn thời gian thật của một System Administrator/DevOps Engineer khi làm việc với Docker trong production: khởi động, dừng, giới hạn tài nguyên, xem log, debug container đang chạy, dọn dẹp disk, và xử lý sự cố lúc 2 giờ sáng khi container liên tục restart. Đây là kỹ năng **Day-2 Operations** — khác hẳn với Day-1 (build image) ở Module 10.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Giới thiệu, bối cảnh Day-2 Operations, mục tiêu học tập |
| [[02-Theory]] | Lifecycle, signal handling, resource limit, restart policy, logging driver — kèm Internal Working |
| [[03-Architecture]] | Sơ đồ ASCII: vòng đời container, luồng signal, kiến trúc resource limit qua cgroups v2 |
| [[04-Commands]] | Toàn bộ lệnh vận hành: lifecycle, resource limit, logs, exec, inspect, events, stats, system df/prune |
| [[05-Labs]] | Bài lab Basic → Intermediate → Advanced/Mini Project |
| [[06-Troubleshooting]] | Restart loop, log làm đầy disk, OOM kill database, "no space left on device" |
| [[07-Interview]] | Câu hỏi phỏng vấn: Junior, Mid-level, Thực chiến |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 12 — Networking |

## Learning Objectives (tổng)

1. Nắm vững toàn bộ vòng đời container (`create` → `start`/`run` → `pause`/`unpause` → `stop`/`kill` → `rm`) và ý nghĩa vận hành của từng trạng thái.
2. Hiểu cơ chế signal handling trong container — vì sao PID 1 trong container đặc biệt, vấn đề zombie process và signal forwarding, và tại sao nhiều ứng dụng "không tắt được sạch" (graceful shutdown) nếu viết Dockerfile sai.
3. Áp đặt và giải thích được resource limit (`--memory`, `--memory-swap`, `--cpus`, `--cpu-shares`, `--cpuset-cpus`) và liên hệ trực tiếp với cgroups v2 đã học ở Module 08 (Linux Sysadmin).
4. Chọn đúng restart policy (`no`, `on-failure`, `always`, `unless-stopped`) cho từng loại workload production, hiểu cơ chế self-healing.
5. Cấu hình logging driver `json-file` với `max-size`/`max-file` để tránh sự cố disk đầy do log — một trong những sự cố production phổ biến nhất với Docker.
6. Phân biệt `docker exec` và `docker attach`, dùng đúng công cụ để debug container đang chạy mà không làm gián đoạn tiến trình chính.
7. Trích xuất thông tin chính xác từ `docker inspect` bằng Go template (`--format`) — kỹ năng bắt buộc khi viết script tự động hóa/giám sát.
8. Sử dụng bộ công cụ giám sát vận hành: `docker events`, `docker stats`, `docker system df`, `docker system prune` — và hiểu rủi ro khi dùng `prune` trong production.

## Yêu cầu tiên quyết

- Đã hoàn thành Linux Sysadmin (29 module) — đặc biệt vững về `systemd`, signal (`kill -l`), permission, filesystem.
- Đã hoàn thành [[../10-Docker-Images/README|Module 10 — Docker Images]] — hiểu image, layer, Dockerfile cơ bản.
- Đã cài Docker Engine (khuyến nghị nhánh 29.x) và có quyền chạy `docker` (user trong group `docker` hoặc dùng `sudo`).

## Công cụ dùng trong module

- Docker Engine CLI (`docker`) — nhánh 29.x.
- Một máy Linux (VM hoặc WSL2) có `systemd` và cgroups v2 (mặc định trên các distro hiện đại — Ubuntu 22.04+, RHEL 9+, Debian 12+).
- Không cần Docker Compose/Kubernetes ở module này — tập trung thuần vào `docker` CLI cấp container đơn lẻ.

> [!note] Self-Review
> - **Accuracy:** Đã WebSearch xác nhận các mốc phiên bản Docker Engine (nhánh 29.x), runc (1.5.x), containerd (2.2.x) tính đến 7/2026; xác nhận hành vi mapping resource flag sang cgroups v2 (`--memory` → `memory.max`, `--memory-reservation` → `memory.low`, `--cpus` → cpu.max quota/period) và ý nghĩa exit code 137 (SIGKILL/OOM) và 143 (SIGTERM) qua tài liệu Docker chính thức. Các con số cấu hình mẫu trong bài (dung lượng RAM, log size...) là ví dụ minh họa, không phải benchmark cố định — được ghi chú rõ trong bài.
> - **Diagram:** Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã được đo bằng script Python (so sánh `len()` của border trên/dưới và mọi dòng nội dung trong từng khung) trước khi đưa vào file — không dùng Mermaid.

## Điều hướng

- Trước đó: [[../10-Docker-Images/README|Module 10 — Docker Images]]
- Tiếp theo: [[../12-Networking/README|Module 12 — Networking]]
