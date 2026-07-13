---
title: "Module 14 - Docker Compose"
tags: [docker, sysops-infra, module-14, compose, orchestration]
module: 14
part: "II - Container Fundamentals"
difficulty: Intermediate
status: draft
created: 2026-07-12
prerequisites: ["[[../13-Storage/README|Module 13]]"]
next: "[[../15-Multi-container-Applications/README|Module 15]]"
---

# Module 14 - Docker Compose

## Vì sao module này quan trọng

Ở Module 09-11, Vinh đã học cách chạy container đơn lẻ bằng `docker run`. Nhưng trong thực tế doanh nghiệp, gần như không có ứng dụng nào chỉ gồm một container. Một hệ thống web bình thường đã cần tối thiểu: web server (nginx), application server (backend), database (postgres/mysql), cache (redis) — 4 container trở lên, mỗi cái có network, volume, biến môi trường, thứ tự khởi động riêng.

Nếu quản lý bằng tay — gõ 4-5 lệnh `docker run` với hàng chục flag mỗi lần, nhớ đúng thứ tự network/volume phải tạo trước — thì chỉ cần một junior gõ sai một flag là môi trường staging "chạy không giống production". Đây chính là lý do Docker Compose ra đời: khai báo TOÀN BỘ hệ thống multi-container trong một file YAML, rồi dùng một lệnh duy nhất (`docker compose up`) để dựng toàn bộ hệ thống đúng như khai báo, lặp lại được (reproducible), review được qua Git (Infrastructure as Code ở quy mô nhỏ).

Đây cũng là bước đệm tư duy trước khi học Kubernetes (thường ở các module sau): Compose dạy Vinh cách nghĩ "khai báo trạng thái mong muốn" (declarative) thay vì "ra lệnh từng bước" (imperative) — chính là triết lý cốt lõi của mọi công cụ orchestration hiện đại.

## Nội dung module

| # | File | Nội dung |
|---|------|----------|
| 1 | [[01-Introduction]] | Compose giải quyết vấn đề gì, mục tiêu học tập |
| 2 | [[02-Theory]] | Compose Specification, cấu trúc file, Internal Working |
| 3 | [[03-Architecture]] | Sơ đồ kiến trúc ASCII: project, network, volume, healthcheck flow |
| 4 | [[04-Commands]] | Toàn bộ lệnh `docker compose ...` + file compose.yaml mẫu đầy đủ |
| 5 | [[05-Labs]] | Lab thực hành Basic -> Intermediate -> Advanced/Mini Project |
| 6 | [[06-Troubleshooting]] | Lỗi thường gặp và cách chẩn đoán |
| 7 | [[07-Interview]] | Câu hỏi phỏng vấn Junior / Mid-level / Thực chiến |
| 8 | [[08-Summary]] | Tóm tắt, cầu nối sang Module 15 |

Thư mục hỗ trợ:
- [[labs/README|labs/]] — nơi lưu code, compose.yaml của từng lab
- `scripts/` — script hỗ trợ tự viết trong quá trình học
- `images/` — hình ảnh minh họa (nếu có, hiện đang trống)

## Mục tiêu sau khi hoàn thành module

Sau module này, Vinh phải tự làm được (không nhìn tài liệu):
- Viết một file `compose.yaml` cho hệ thống 3-4 service (web, app, db, cache) có network, volume, healthcheck, `depends_on` đúng chuẩn.
- Giải thích được Compose tự tạo network/volume/tên container theo quy tắc nào (project name).
- Dùng `profiles` để tách service "chỉ chạy khi cần" (ví dụ debug tool, admin UI) ra khỏi luồng chạy mặc định.
- Quản lý biến môi trường qua `.env` đúng thứ tự ưu tiên, không hardcode secret vào compose.yaml.
- Debug được các lỗi Compose thường gặp: container khởi động trước khi dependency sẵn sàng, `.env` không được load, port bị chiếm, thay đổi compose.yaml không có tác dụng.

## Điều kiện tiên quyết

- Đã hoàn thành 29 module Linux Sysadmin (SSH, systemd, networking, permission, filesystem, security hardening).
- Đã học Module 09-Docker-Engine, Module 11-Container-Runtime (hiểu container, image, network, volume ở mức Docker Engine đơn lẻ).
- Đang học song song Ansible — sẽ thấy nhiều điểm tương đồng về tư duy "khai báo trạng thái mong muốn" giữa Compose và Ansible playbook.

> [!note] Self-Review
> Đã dùng WebSearch (12/07/2026) để xác nhận: (1) Docker Compose CLI nhánh hiện hành là 5.x, bản mới nhất tại thời điểm viết là v5.3.1 (phát hành 07/07/2026), là Go binary chạy như Docker CLI plugin (`docker compose`, không dấu gạch ngang); (2) khóa top-level `version:` trong compose.yaml đã bị Compose bỏ qua (ignored) từ Compose v2 trở đi và được xác nhận là obsolete trên tài liệu chính thức Docker Docs — do đó toàn bộ ví dụ trong module này KHÔNG dùng khóa `version:`, đặt tên file `compose.yaml`; (3) cú pháp `healthcheck` (test/interval/timeout/retries/start_period), `depends_on.condition: service_healthy`, `profiles`, `secrets` (file-based, mount vào `/run/secrets/<name>`), và thứ tự ưu tiên biến môi trường (shell/`-e` > `environment` có interpolation > `environment` trong file > `env_file` > `ENV` trong image) đều đã được đối chiếu với Docker Docs chính thức trước khi đưa vào bài viết.
> Toàn bộ sơ đồ ASCII trong [[03-Architecture]] (3 sơ đồ) đã được đo bằng script Python (so khớp `len()` của border trên/dưới/nội dung từng khung) trước khi đưa vào file — tất cả đều cho kết quả OK, không còn khung nào lệch độ rộng.
