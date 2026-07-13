---
title: "Scripts - Module 13"
module: 13
tags: [docker, sysops-infra, module-13, scripts]
---

# Scripts - Module 13: Storage

Script hỗ trợ backup/audit volume, dùng kèm [[../04-Commands|04-Commands.md]].

## Script gợi ý tự viết

| Script | Mục đích |
|---|---|
| `backup-volume.sh` | Script backup định kỳ hoàn chỉnh (phát triển từ Lab 6): nén volume, dọn backup cũ theo số lượng giữ lại, kiểm tra toàn vẹn archive bằng `tar tzf` |
| `restore-volume.sh` | Script restore đối xứng với `backup-volume.sh`, nhận tên file backup và tên volume đích |
| `anonymous-volume-audit.sh` | Liệt kê toàn bộ volume có tên dạng hash ngẫu nhiên (dấu hiệu anonymous volume), đối chiếu container nào đang tham chiếu để đánh giá có an toàn dọn dẹp hay không |
| `bind-mount-permission-check.sh` | Nhận đường dẫn bind mount và tên image, so sánh UID/GID của tiến trình trong image với ownership thư mục host, cảnh báo sớm nguy cơ `permission denied` trước khi chạy container thật |

## Quy ước

- Script bash, `set -euo pipefail`.
- `backup-volume.sh`/`restore-volume.sh` luôn mount volume nguồn ở chế độ `:ro` khi backup — không có ngoại lệ.
- Không script nào tự động `docker volume rm` mà không có bước xác nhận thủ công hoặc dry-run trước, theo đúng bài học ở [[../06-Troubleshooting|06-Troubleshooting.md]] Sự cố 3.
