---
title: "Module 15 - Multi-container Applications"
tags: [docker, sysops-infra, module-15, compose, capstone]
module: 15
part: "II - Container Fundamentals"
difficulty: Intermediate
status: draft
created: 2026-07-12
prerequisites: ["[[../14-Docker-Compose/README|Module 14]]"]
next: "[[../16-Image-Registry/README|Module 16]]"
---

# Module 15 - Multi-container Applications

## Vì sao module này là điểm kết của Part II

Bảy module vừa qua (08-14) mỗi module dạy một mảnh kiến thức riêng: kernel internals, kiến trúc Docker Engine, build image, vận hành container, networking, storage, Compose. Module 15 không dạy khái niệm mới — nó **ghép toàn bộ 7 mảnh đó thành một hệ thống thật**, đúng những gì bạn sẽ làm trong công việc: dựng một stack web application hoàn chỉnh gồm Nginx (reverse proxy), Web/App backend, Database (MySQL/Postgres), Redis (cache), và RabbitMQ (message queue) — 5 loại thành phần xuất hiện trong gần như mọi kiến trúc microservice/web application doanh nghiệp hiện đại.

Đây là bài kiểm tra thực sự: nếu bạn có thể tự tay dựng stack này từ đầu, giải thích được từng quyết định thiết kế (vì sao network tách biệt thế này, vì sao volume đặt ở đây, vì sao healthcheck cấu hình vậy), bạn đã sẵn sàng cho Module 16 (Image Registry) và các khóa Kubernetes tiếp theo — vì Kubernetes về bản chất giải quyết đúng bài toán này ở quy mô nhiều node.

## Nội dung

| # | File | Nội dung chính |
|---|------|-----------------|
| 1 | [[01-Introduction]] | Bài toán tổng hợp: vì sao cần 5 thành phần này, use case thật |
| 2 | [[02-Theory]] | Vai trò từng thành phần, cách chúng phối hợp, Internal Working |
| 3 | [[03-Architecture]] | Sơ đồ ASCII: kiến trúc toàn stack, luồng request, phân vùng network |
| 4 | [[04-Commands]] | compose.yaml hoàn chỉnh, lệnh vận hành stack |
| 5 | [[05-Labs]] | Lab dựng stack từ cơ bản đến mini project hoàn chỉnh |
| 6 | [[06-Troubleshooting]] | Sự cố thực tế khi vận hành stack nhiều thành phần |
| 7 | [[07-Interview]] | Câu hỏi phỏng vấn Junior/Mid-level/Thực chiến |
| 8 | [[08-Summary]] | Tổng kết Part II, cầu nối sang Module 16 |
| - | [[labs/README|Labs folder]] | Hướng dẫn môi trường thực hành |
| - | [[scripts/README|Scripts folder]] | Script hỗ trợ |

## Mục tiêu học xong module

- Giải thích được vai trò riêng biệt của Nginx reverse proxy, backend, database, Redis, RabbitMQ trong một kiến trúc web application thật.
- Tự viết được một `compose.yaml` production-grade ghép cả 5 thành phần, với network phân vùng đúng nguyên tắc bảo mật, volume cho dữ liệu bền vững, healthcheck và `depends_on` đúng thứ tự.
- Giải thích được vì sao Redis và RabbitMQ tuy đều có thể "làm trung gian" nhưng giải quyết hai bài toán khác nhau (cache/session vs message queue bất đồng bộ).
- Vận hành được toàn bộ vòng đời stack: dựng, kiểm tra sức khỏe, xem log tổng hợp, cập nhật rolling từng service, backup dữ liệu, dừng an toàn.
- Debug được các sự cố đặc thù khi nhiều thành phần phối hợp: race condition lúc khởi động, deadlock kết nối, message queue tồn đọng.

## Điều kiện tiên quyết

- [[../14-Docker-Compose/README|Module 14]] — thành thạo compose.yaml, healthcheck, depends_on, network, volume, secrets.
- Toàn bộ kiến thức Module 08-13 (namespace/cgroup, Docker Engine, Dockerfile, container runtime, networking, storage) — module này không nhắc lại lý thuyết nền, chỉ áp dụng.

## Tiếp theo

Sau khi hoàn thành module này — cũng là module cuối cùng của **Part II - Container Fundamentals** — sang [[../16-Image-Registry/README|Module 16 - Image Registry]] để học cách quản lý vòng đời image ở quy mô tổ chức: registry nội bộ, versioning, scanning bảo mật, cleanup policy.

> [!note] Self-Review
> - Vai trò và cấu hình của từng thành phần (Nginx reverse proxy, MySQL/Postgres, Redis, RabbitMQ) đối chiếu với facts sheet đã xác nhận trước và kiến thức đã thống nhất xuyên suốt Module 09-14 (Docker Engine 29.x, Compose CLI 5.x, không dùng khóa `version:`, cgroups v2 mặc định) — không phát sinh số liệu mới ngoài facts sheet.
> - Toàn bộ sơ đồ ASCII trong [[03-Architecture]] đã được đo bằng script Python (`len()` từng dòng khớp border trên/dưới) trước khi ghi file.
