---
title: "Labs - Module 15"
module: 15
tags: [docker, sysops-infra, module-15, labs]
---

# Labs - Module 15: Multi-container Applications

Thư mục lưu toàn bộ source code, compose.yaml, và tài liệu vận hành từ [[../05-Labs|05-Labs.md]] — đây là bài tập lớn tổng hợp cuối Part II.

## Môi trường

- VM Linux đã cài Docker Engine 29.x kèm Compose plugin nhánh 5.x.
- Khuyến nghị tối thiểu 4GB RAM cho VM lab để chạy đồng thời 6 container (nginx, backend, worker, db, redis, rabbitmq).
- Đủ dung lượng đĩa cho image `postgres`, `redis`, `rabbitmq:4-management-alpine` (RabbitMQ management image nặng hơn bản thường).

## File/thư mục dự kiến sinh ra trong thư mục này

| Thư mục/File | Sinh ra ở Lab | Nội dung |
|---|---|---|
| `backend/` | Lab 2 | Source code + Dockerfile backend tối giản |
| `worker/` | Lab 5 | Source code + Dockerfile worker tiêu thụ RabbitMQ |
| `nginx/nginx.conf` | Lab 3 | Cấu hình reverse proxy trỏ đúng backend |
| `compose.yaml`, `.env.example` | Lab 3, Lab 6 | File Compose hoàn chỉnh của stack |
| `secrets/` | Lab 3, Lab 6 | File secret mẫu (kèm `.gitignore`, không chứa giá trị thật) |
| `race-condition-evidence.md` | Lab 4 | Log bằng chứng có/không healthcheck |
| `runbook.md` | Lab 6 (mini project) | Tài liệu vận hành: dựng, backup, rolling update, xử lý sự cố |

## Lưu ý an toàn

- Không commit `secrets/*.txt` hay `.env` chứa giá trị thật vào Git — chỉ commit file `.example`.
- Lab 6 mô phỏng sự cố (`docker compose kill redis`) — chỉ thực hiện trên VM lab, không thực hiện trên bất kỳ môi trường nào có traffic thật.
