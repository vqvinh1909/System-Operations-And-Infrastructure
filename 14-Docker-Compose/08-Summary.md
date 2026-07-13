---
title: "08 - Summary"
module: 14
tags: [docker, sysops-infra, module-14, summary]
---

# 08 - Summary: Docker Compose

## Những gì đã học

- **Compose Specification**: chuẩn hợp nhất hiện tại, khóa `version:` đã lỗi thời/bị bỏ qua, dùng tên file `compose.yaml`.
- **Services**: `build` (dev) vs `image` (production, đã build sẵn từ CI/CD registry), `depends_on` short syntax (chỉ chờ start) vs long syntax `condition: service_healthy` (chờ thực sự sẵn sàng) — đây là cái bẫy phổ biến nhất.
- **Networks**: default network tự tạo với DNS nội bộ theo tên service; custom network để tách biệt bảo mật (frontend/backend); external network để chia sẻ giữa nhiều project.
- **Volumes**: named volume (Docker quản lý, sống sót qua `docker compose down` không `-v`) vs bind mount (Compose không sở hữu, không bị `down -v` xóa).
- **Profiles**: tách service phụ trợ (debug/admin tool) ra khỏi luồng mặc định, kích hoạt qua `--profile` hoặc `COMPOSE_PROFILES`.
- **Environment & `.env`**: thứ tự ưu tiên từ CLI (cao nhất) → interpolation từ `.env`/shell → `environment` hardcode → `env_file` → `ENV` trong image (thấp nhất).
- **Secrets**: file-based, mount vào `/run/secrets/<name>`, không hardcode password vào `environment:`.
- **Healthcheck**: `test`/`interval`/`timeout`/`retries`/`start_period`, liên kết trực tiếp với `depends_on.condition: service_healthy`.
- **Internal Working**: Compose không có daemon riêng, chỉ gọi Docker Engine API; project name là tiền tố mọi resource; recreate container dựa vào so sánh hash cấu hình.

## Checklist tự đánh giá

- [ ] Viết được compose.yaml cho stack 4+ service với network tách biệt, volume, healthcheck, secrets, profiles đúng chuẩn.
- [ ] Giải thích chính xác vì sao `depends_on` short syntax không đủ để tránh crash loop lúc khởi động lần đầu.
- [ ] Debug được lỗi biến `.env` không nội suy, lỗi container không recreate sau khi sửa compose.yaml.
- [ ] Phân biệt rõ hệ quả `docker compose down` vs `down -v`, không bao giờ nhầm lẫn trên production.
- [ ] Dùng thành thạo `docker compose config` để kiểm tra cấu hình cuối cùng trước khi deploy.

## Liên kết tới Module tiếp theo

Module này đã trang bị đủ công cụ để khai báo một stack multi-container hoàn chỉnh. [[../15-Multi-container-Applications/README|Module 15 - Multi-container Applications]] là module tổng hợp cuối Part II — áp dụng toàn bộ kiến thức từ Module 08 đến 14 (namespace/cgroup, Docker Engine, Dockerfile, runtime, networking, storage, Compose) để dựng một stack thực tế hoàn chỉnh: Web (Nginx) + Backend + Database (MySQL/Postgres) + Redis + RabbitMQ + Nginx Reverse Proxy, ghép thành một hệ thống production-grade bằng Docker Compose.

> [!tip] Ghi nhớ cốt lõi
> Compose không phải công cụ mới — nó là lớp khai báo (declarative) mỏng gọi lại đúng Docker Engine API bạn đã học ở Module 09-13. Hiểu rõ từng khái niệm bên dưới (network, volume, healthcheck, container lifecycle) khiến việc đọc và viết compose.yaml trở thành phản xạ, không phải học thuộc cú pháp YAML một cách máy móc.
