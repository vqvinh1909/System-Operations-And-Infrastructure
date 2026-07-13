---
title: "Module 12 - Networking"
tags: [docker, sysops-infra, module-12, networking, bridge]
module: 12
part: "II - Container Fundamentals"
difficulty: Intermediate
status: draft
created: 2026-07-12
prerequisites: ["[[../11-Container-Runtime/README|Module 11]]"]
next: "[[../13-Storage/README|Module 13]]"
---

# Module 12 - Networking

## Networking là gì trong bối cảnh Docker

Ở khóa Linux Sysadmin, Vinh đã học network namespace, bridge, iptables/nftables, routing ở mức hệ điều hành. Module này không dạy lại networking từ đầu — nó cho Vinh thấy **Docker không phát minh ra công nghệ mạng mới**. Docker chỉ là một lớp "đạo diễn" (orchestration layer) gọi lại đúng những công cụ Linux Vinh đã biết: `ip netns`, Linux bridge, `veth` pair, iptables/nftables NAT — rồi tự động hoá chúng mỗi khi có lệnh `docker run` hay `docker network create`.

Hiểu rõ tầng dưới này là ranh giới giữa một người "biết gõ lệnh Docker" và một System Administrator thực sự troubleshoot được khi container không ra Internet, hai container không thấy nhau, hoặc port bị "already allocated" lúc 2 giờ sáng.

## Vì sao module này quan trọng trong công việc thực tế

- Khi container A không gọi được container B bằng tên — lỗi networking phổ biến nhất khi triển khai multi-container app (ví dụ app + database) trên một host, đặc biệt khi ai đó vẫn dùng default bridge theo thói quen cũ.
- Khi deploy production và gặp lỗi `port is already allocated` — cần hiểu Docker quản lý port binding ở đâu để xử lý nhanh, không phải reboot server.
- Khi cần cô lập container ra khỏi mạng nội bộ (network driver `none`) cho các job xử lý dữ liệu nhạy cảm không cần network.
- Khi công ty cần container xuất hiện như một thiết bị vật lý riêng trên VLAN (macvlan) — ví dụ ứng dụng legacy cần IP cố định trong dải mạng doanh nghiệp, không thể dùng NAT.
- Khi làm việc với Docker Swarm (overlay network) — dù khóa này không đi sâu Swarm, biết khái niệm VXLAN giúp Vinh không bị bỡ ngỡ khi đọc kiến trúc multi-host sau này (Kubernetes CNI cũng dùng overlay tương tự).

## Nội dung module

| # | File | Nội dung |
|---|------|----------|
| 01 | [[01-Introduction]] | Giới thiệu, mục tiêu học tập |
| 02 | [[02-Theory]] | Lý thuyết chi tiết: network namespace, veth pair, bridge, iptables NAT, 5 network driver, DNS |
| 03 | [[03-Architecture]] | Sơ đồ kiến trúc ASCII: luồng gói tin qua bridge + NAT, bảng so sánh driver, luồng DNS resolution |
| 04 | [[04-Commands]] | Cú pháp `docker network create/ls/inspect/connect/disconnect/rm`, `-p`, `--network` |
| 05 | [[05-Labs]] | Lab thực hành: bridge mặc định vs user-defined, port mapping, macvlan, mini project |
| 06 | [[06-Troubleshooting]] | Lỗi thực tế: không resolve tên nhau, port allocated, không ra Internet, macvlan không ping host |
| 07 | [[07-Interview]] | Câu hỏi phỏng vấn: Junior, Mid-level, Thực chiến |
| 08 | [[08-Summary]] | Tóm tắt, cầu nối sang Module 13 - Storage |

## Thư mục hỗ trợ

- [[labs/README|labs/]] — nơi lưu code/config khi thực hành lab
- [[scripts/README|scripts/]] — nơi lưu script tự viết trong quá trình học
- `images/` — hình ảnh minh hoạ (nếu có, bổ sung sau)

## Điều kiện tiên quyết

- Đã hoàn thành [[../11-Container-Runtime/README|Module 11 - Container Runtime]]: hiểu container là gì, container runtime hoạt động ra sao.
- Đã học xong Linux Sysadmin cơ bản: network namespace, iptables/nftables, bridge, routing.
- Đã cài Docker Engine (nhánh 29.x) và có quyền chạy `docker` (user trong group `docker` hoặc dùng `sudo`).

## Bước tiếp theo

Sau module này, Vinh chuyển sang [[../13-Storage/README|Module 13 - Storage]] — tìm hiểu volume, bind mount, cách Docker quản lý dữ liệu bền vững cho container.

> [!note] Self-Review
> - Đã dùng WebSearch xác nhận (12/07/2026): cơ chế Docker embedded DNS server tại 127.0.0.11 chỉ hoạt động trên user-defined network (default bridge không có); đặc điểm driver macvlan (container có IP/MAC riêng trên LAN, không qua NAT, container không ping được host theo giới hạn kernel Linux); overlay network dùng VXLAN, yêu cầu Swarm mode, cổng 4789/UDP cho data plane, 7946 TCP/UDP cho discovery, 2377 TCP cho quản lý cluster; khác biệt `-p`/`--publish` (mở port ra ngoài host qua iptables DNAT) và `--expose`/`EXPOSE` (chỉ mang tính documentation, không mở port ra ngoài).
> - Đã viết sơ đồ ASCII nháp và đo bằng script Python (đếm độ dài từng dòng trong nhóm liền kề bắt đầu bằng `+`/`|`) cho cả 3 sơ đồ trong [[03-Architecture]] — tất cả đều cho kết quả `OK`, không còn `MISMATCH`.
