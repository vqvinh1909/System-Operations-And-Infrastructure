---
title: "Scripts - Module 14"
module: 14
tags: [docker, sysops-infra, module-14, scripts]
---

# Scripts - Module 14: Docker Compose

Script hỗ trợ vận hành Compose, dùng kèm [[../04-Commands|04-Commands.md]].

## Script gợi ý tự viết

| Script | Mục đích |
|---|---|
| `compose-config-diff.sh` | Chạy `docker compose config` trước và sau khi sửa file, diff kết quả để xác nhận đúng thay đổi trước khi `up` thật |
| `safe-compose-down.sh` | Wrapper an toàn cho `docker compose down` — cảnh báo rõ ràng và yêu cầu xác nhận nếu phát hiện cờ `-v` trong tham số, theo bài học ở [[../06-Troubleshooting|06-Troubleshooting.md]] Sự cố 4 |
| `secret-file-init.sh` | Sinh sẵn cấu trúc thư mục `secrets/` với file `.example` và `.gitignore` chuẩn cho project Compose mới, tránh quên bảo vệ secret ngay từ đầu |
| `healthcheck-timing-report.sh` | Chạy `docker compose up -d` và đo thời gian từng service đạt `healthy`, hữu ích khi tối ưu thứ tự khởi động (liên hệ câu hỏi thực chiến 14 ở [[../07-Interview|07-Interview.md]]) |

## Quy ước

- Script bash, `set -euo pipefail`.
- `safe-compose-down.sh` là script quan trọng nhất về an toàn dữ liệu — không bao giờ cho phép `-v` chạy tự động không xác nhận trên môi trường được đánh dấu production.
- Mọi script tạo file secret mẫu phải luôn kèm `.gitignore` tương ứng, không để lọt secret thật vào Git dù chỉ trong lúc test.
