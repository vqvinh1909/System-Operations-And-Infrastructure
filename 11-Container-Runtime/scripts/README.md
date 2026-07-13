---
title: "Scripts - Module 11"
module: 11
tags: [docker, sysops-infra, module-11, scripts]
---

# Scripts - Module 11: Container Runtime

Script hỗ trợ vận hành hàng ngày, dùng kèm [[../04-Commands|04-Commands.md]].

## Script gợi ý tự viết

| Script | Mục đích |
|---|---|
| `oom-audit.sh` | Quét toàn bộ container đang chạy, báo cáo container nào từng bị `OOMKilled: true` gần nhất và `RestartCount` hiện tại |
| `log-size-audit.sh` | Liệt kê top 10 container có file log lớn nhất, cảnh báo nếu vượt ngưỡng tự chọn |
| `safe-prune.sh` | Wrapper cho `docker system prune` — luôn in `docker system df -v` trước, yêu cầu xác nhận thủ công (`read -p`), mặc định KHÔNG bật `--volumes` trừ khi truyền cờ rõ ràng |
| `restart-policy-report.sh` | Liệt kê toàn bộ container kèm restart policy hiện tại — dùng rà soát xem service nào đang thiếu cấu hình phù hợp |

## Quy ước

- Script bash, `set -euo pipefail`.
- `safe-prune.sh` là script quan trọng nhất về mặt an toàn — không bao giờ để mặc định chạy `--volumes` mà không có xác nhận rõ ràng, theo đúng bài học ở [[../06-Troubleshooting|06-Troubleshooting.md]] Sự cố 4.
- Mọi script đọc số liệu (`docker stats`, `docker system df`) nên hỗ trợ output dạng có thể ghi log/cron, không chỉ chạy tương tác.
