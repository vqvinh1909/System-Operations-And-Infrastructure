---
title: "03 - Architecture"
module: 11
tags: [docker, sysops-infra, module-11, architecture, diagram]
---

# 03 - Architecture: Lifecycle, Signal Flow, Resource Limit

Toàn bộ sơ đồ dưới đây đã được đo bằng script Python (so sánh `len()` border trên/dưới và mọi dòng nội dung trong từng khung) trước khi đưa vào file.

## 1. Sơ đồ vòng đời container

```text
+----------------------------------------------------------------+
|                      CONTAINER LIFECYCLE                       |
+----------------------------------------------------------------+
|        CREATED   (docker create - chua chay tien trinh)        |
+----------------------------------------------------------------+
|  RUNNING   (docker start / docker run - tien trinh dang chay)  |
+----------------------------------------------------------------+
|    PAUSED    (docker pause - dong bang bang cgroup freezer)    |
+----------------------------------------------------------------+
| EXITED    (docker stop/kill - tien trinh da dung, con ton tai) |
+----------------------------------------------------------------+
|      REMOVED   (docker rm - xoa hoan toan khoi he thong)       |
+----------------------------------------------------------------+
```

Đọc từ trên xuống theo đúng thứ tự vòng đời tự nhiên. Lưu ý: `PAUSED` chỉ là trạng thái tạm, luôn quay lại `RUNNING` qua `docker unpause`, không có đường đi trực tiếp từ `PAUSED` sang `REMOVED` — phải `unpause` về `RUNNING` rồi mới `stop`/`kill` để tới `EXITED`. `EXITED` là trạng thái container **vẫn còn tồn tại** trên đĩa (writable layer, log, exit code) — chỉ `docker rm` mới thực sự xóa.

## 2. Sơ đồ luồng signal: có và không có `--init`

```text
KHONG CO --init                          CO --init (dung tini)

PID 1 = ung dung cua ban                 PID 1 = tini (init nhe)
   |                                        |
   | SIGTERM co the bi bo qua               | nhan SIGTERM, forward dung
   | (neu khong tu dang ky handler)         v xuong tien trinh con
   v                                     PID con = ung dung cua ban
(khong reap zombie con neu co fork)         |
                                             | ung dung xu ly signal
                                             | binh thuong nhu tren host
                                         tini tu dong reap zombie
```

So sánh trực tiếp: không có `--init`, ứng dụng của bạn là PID 1 thật, chịu quy tắc đặc biệt của kernel (không tự áp dụng hành vi mặc định cho signal nếu tiến trình không tự đăng ký handler) — dễ dẫn tới container "không chịu dừng" khi `docker stop`. Có `--init`, `tini` đóng vai trò PID 1, forward signal đúng chuẩn xuống ứng dụng (giờ chỉ là tiến trình con bình thường) và tự động reap zombie process — loại bỏ cả hai vấn đề đã phân tích ở [[02-Theory]] mục 2.

## 3. Sơ đồ mapping flag Docker sang cgroups v2

```text
+------------------------------------------------------+
|         MAPPING DOCKER FLAG SANG CGROUPS V2          |
+------------------------------------------------------+
| --memory        ->  memory.max  (gioi han cung RAM)  |
+------------------------------------------------------+
|  --memory-reservation  ->  memory.low  (nguong mem)  |
+------------------------------------------------------+
|    --cpus          ->  cpu.max  (quota / period)     |
+------------------------------------------------------+
| --cpu-shares    ->  cpu.weight  (trong so tuong doi) |
+------------------------------------------------------+
| --cpuset-cpus   ->  cpuset.cpus  (ghim core cu the)  |
+------------------------------------------------------+
```

Đây chính là bằng chứng trực quan cho nguyên lý đã học ở Module 08: Docker không tự phát minh cơ chế giới hạn tài nguyên, nó chỉ dịch flag CLI thành thao tác ghi file ảo trong `/sys/fs/cgroup/...`. Khi container vượt `memory.max`, kernel OOM Killer kích hoạt trong đúng phạm vi cgroup đó — cô lập sự cố vào một container, không ảnh hưởng toàn bộ host.

## 4. Luồng OOM Kill — từ vượt giới hạn tới exit code 137

```text
+-----------------------------------+
| Container cap phat RAM vuot       |
| memory.max (--memory da dat)      |
+-----------------------------------+
              |
              v
+-----------------------------------+
| Kernel OOM Killer kich hoat       |
| (pham vi: dung cgroup container)  |
+-----------------------------------+
              |
              v
+-----------------------------------+
| Tien trinh chinh bi SIGKILL       |
| (khong graceful, khong flush)     |
+-----------------------------------+
              |
              v
+-----------------------------------+
| Container: EXITED, ExitCode=137   |
| State.OOMKilled = true            |
+-----------------------------------+
              |
              v
+-----------------------------------+
| Neu co restart policy phu hop:    |
| Docker tu khoi dong lai container |
+-----------------------------------+
```

Sơ đồ này là chuỗi nhân quả bạn cần nhớ để chẩn đoán đúng khi thấy exit code 137: luôn xác nhận bằng `docker inspect --format '{{.State.OOMKilled}}'` thay vì chỉ suy đoán từ exit code (vì `docker kill` cũng cho ra 137 nhưng không phải do OOM).

Tiếp theo: [[04-Commands]] để xem cú pháp lệnh cụ thể áp dụng toàn bộ khái niệm ở [[02-Theory]] và sơ đồ trên.
