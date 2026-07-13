---
title: "Scripts - Module 12"
module: 12
tags: [docker, sysops-infra, module-12, scripts]
---

# Scripts - Module 12: Networking

Script hỗ trợ chẩn đoán network, dùng kèm [[../04-Commands|04-Commands.md]].

## Script gợi ý tự viết

| Script | Mục đích |
|---|---|
| `network-audit.sh` | Liệt kê toàn bộ network hiện có, container nào thuộc network nào, cảnh báo nếu có container quan trọng vẫn nằm trên default bridge |
| `port-conflict-check.sh` | Nhận một port làm tham số, kiểm tra cả `docker ps --filter publish=` lẫn `ss -tlnp` trên host để xác định thủ phạm đang chiếm port trước khi `docker run -p` |
| `dnat-snapshot.sh` | Chụp lại toàn bộ rule `iptables -t nat -L DOCKER -n` kèm timestamp — hữu ích so sánh trước/sau khi restart Docker daemon hoặc bật firewall mới theo Sự cố 6 ở [[../06-Troubleshooting|06-Troubleshooting.md]] |

## Quy ước

- Script bash, `set -euo pipefail`.
- Mọi script chạy `iptables`/`sysctl` chỉ ở chế độ đọc (list/audit), không tự động sửa cấu hình firewall production mà không có xác nhận thủ công.
- Ghi rõ comment giải thích mapping giữa kết quả script và khái niệm lý thuyết tương ứng (namespace, veth, DNAT, MASQUERADE) để dùng làm tài liệu ôn tập nhanh.
