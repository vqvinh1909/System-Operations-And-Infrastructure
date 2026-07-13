---
title: "Scripts - Module 08"
module: 8
tags: [docker, sysops-infra, module-08, scripts]
---

# Scripts - Module 08: Container Technology

Thư mục này chứa script hỗ trợ tra cứu/kiểm tra nhanh liên quan namespace và cgroup, dùng kèm [[../04-Commands|04-Commands.md]].

## Script gợi ý tự viết

| Script | Mục đích |
|---|---|
| `ns-inspect.sh` | Nhận PID làm tham số, in ra toàn bộ namespace (`/proc/<pid>/ns/`) và inode number tương ứng, tiện so sánh nhanh hai tiến trình có chung namespace hay không |
| `cgroup-usage.sh` | Nhận đường dẫn cgroup, in gọn `memory.current`, `memory.max`, `cpu.stat` (nr_throttled), `pids.current` — dùng khi chẩn đoán OOM-kill/CPU throttling ở [[../06-Troubleshooting|06-Troubleshooting.md]] |
| `overlay-demo-cleanup.sh` | Dọn dẹp nhanh thư mục thử nghiệm OverlayFS thủ công (`umount` + `rm -rf`) sau khi làm xong phần 5 của `04-Commands.md`, tránh quên umount gây rác |

## Quy ước

- Script viết bằng `bash`, có `set -euo pipefail` ở đầu để fail nhanh khi gõ sai đường dẫn `/sys/fs/cgroup`.
- Không script nào được chạy mặc định với quyền ghi vào cgroup hệ thống (`/sys/fs/cgroup` gốc) — chỉ thao tác trong cgroup con tự tạo (`.../demo-group`, `.../lab-cgroup`...).
- Đặt comment giải thích rõ mỗi lệnh tương ứng cơ chế kernel nào (namespace loại gì, cgroup controller nào) — mục tiêu của module là hiểu bản chất, không chỉ chạy được lệnh.
