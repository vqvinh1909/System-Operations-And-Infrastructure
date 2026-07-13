---
title: "Scripts - Module 09"
module: 9
tags: [docker, sysops-infra, module-09, scripts]
---

# Scripts - Module 09: Docker Engine

Script hỗ trợ tra cứu nhanh liên quan Docker Engine, dùng kèm [[../04-Commands|04-Commands.md]].

## Script gợi ý tự viết

| Script | Mục đích |
|---|---|
| `daemon-health-check.sh` | Kiểm tra nhanh `systemctl status docker`, `docker info`, socket permission, cảnh báo nếu cgroup driver không phải `systemd` |
| `docker-group-audit.sh` | Liệt kê toàn bộ user thuộc nhóm `docker` trên hệ thống — dùng định kỳ rà soát bảo mật theo rủi ro đã học ở [[../02-Theory|02-Theory.md]] mục 4 |
| `container-lifecycle-watch.sh` | Theo dõi `docker events` để log lại mọi lần container start/stop/die/oom kèm timestamp, hữu ích khi điều tra sự cố container tự restart |

## Quy ước

- Script bash, `set -euo pipefail`.
- Không script nào tự động `usermod -aG docker` mà không có xác nhận thủ công (`read -p "Confirm? [y/N]"`) — tránh vô tình cấp quyền root-equivalent hàng loạt qua automation.
- Ghi rõ comment: script nào an toàn chạy trên production (chỉ đọc, ví dụ audit) và script nào chỉ nên chạy trên VM lab (ví dụ script mô phỏng Lab 5).
