---
title: "Scripts - Module 15"
module: 15
tags: [docker, sysops-infra, module-15, scripts]
---

# Scripts - Module 15: Multi-container Applications

Script hỗ trợ vận hành toàn bộ stack, tổng hợp kỹ năng từ Module 09-14, dùng kèm [[../04-Commands|04-Commands.md]].

## Script gợi ý tự viết

| Script | Mục đích |
|---|---|
| `stack-health-check.sh` | Chạy healthcheck thủ công cho cả 5 thành phần (`pg_isready`, `redis-cli ping`, `rabbitmq-diagnostics ping`, `curl` endpoint `/health` của backend, `curl` Nginx), tổng hợp báo cáo một lần thay vì kiểm tra riêng lẻ |
| `backup-full-stack.sh` | Backup có kiểm soát: dùng `pg_dump` cho database (không nén thô như Module 13, đúng bài học Sự cố 5 ở [[../06-Troubleshooting|06-Troubleshooting.md]]), lưu kèm timestamp và dọn bản cũ |
| `scale-worker.sh` | Nhận số lượng worker mong muốn làm tham số, gọi `docker compose up -d --scale worker=N`, kèm kiểm tra độ dài queue RabbitMQ trước/sau để đánh giá hiệu quả scale |
| `queue-depth-monitor.sh` | Định kỳ gọi RabbitMQ management API lấy độ dài từng queue, cảnh báo nếu vượt ngưỡng tự chọn — mô phỏng đơn giản của autoscaling logic sẽ học đầy đủ ở Kubernetes |

## Quy ước

- Script bash, `set -euo pipefail`.
- `backup-full-stack.sh` không bao giờ dùng `tar` nén trực tiếp thư mục dữ liệu Postgres khi database đang chạy — luôn dùng `pg_dump` để đảm bảo tính nhất quán logic.
- Mọi script gọi RabbitMQ management API phải đọc credential từ file secret, không hardcode password trong script.
