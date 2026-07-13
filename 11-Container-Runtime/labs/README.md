---
title: "Labs - Module 11"
module: 11
tags: [docker, sysops-infra, module-11, labs]
---

# Labs - Module 11: Container Runtime

Thư mục lưu script/log thực nghiệm từ [[../05-Labs|05-Labs.md]].

## Môi trường

- VM Linux đã cài Docker Engine 29.x, cgroups v2, quyền `sudo`.
- Tùy chọn: Python 3 để viết script test OOM/signal ở Lab 2, Lab 3.

## File dự kiến sinh ra trong thư mục này

| File | Sinh ra ở Lab | Nội dung |
|---|---|---|
| `stop-vs-kill-timing.md` | Lab 1 | Bảng thời gian đo được của `stop`/`kill` |
| `worker.py`, `zombie-comparison.md` | Lab 2 | Script test zombie, bảng so sánh có/không `--init` |
| `oom-test.py`, `oom-evidence.md` | Lab 3 | Script cấp phát RAM tăng dần, bằng chứng OOM kill (dmesg, docker inspect) |
| `restart-loop-notes.md` | Lab 4 | Ghi chú số lần restart theo từng policy |
| `log-rotation-verify.md` | Lab 5 | Bằng chứng dung lượng log trước/sau khi cấu hình `max-size`/`max-file` |
| `mini-dashboard.sh` | Lab 6 | Script giám sát tổng hợp `docker events`/`docker stats`/`docker system df` |

## Lưu ý an toàn

- Lab 3 (OOM test) có thể làm chậm VM tạm thời trong lúc script cấp phát RAM chạy — nên thực hiện trên VM lab riêng, theo dõi sát bằng terminal thứ hai để dừng kịp thời nếu ảnh hưởng tới cả hệ thống.
- Không chạy `docker system prune -a --volumes` trong lúc làm lab nếu VM đang có dữ liệu lab khác cần giữ lại.
