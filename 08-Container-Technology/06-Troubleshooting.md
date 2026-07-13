---
title: "06 - Troubleshooting"
module: 8
tags: [docker, sysops-infra, module-08, troubleshooting]
---

# 06 - Troubleshooting: Sự cố thật liên quan Namespace/Cgroup/OverlayFS

## Sự cố 1: "Container không tắt được bằng `docker stop`, phải `docker kill`"

**Triệu chứng**: `docker stop <container>` treo 10 giây rồi Docker tự động chuyển sang `SIGKILL`.

**Nguyên nhân gốc**: `docker stop` gửi `SIGTERM` tới tiến trình **PID 1 bên trong PID namespace** của container. Nếu ứng dụng của bạn không chạy trực tiếp làm PID 1 (ví dụ Dockerfile dùng `CMD myapp` nhưng shell `/bin/sh -c` mới là PID 1, còn `myapp` là con), hoặc ứng dụng không có handler xử lý `SIGTERM`, tín hiệu không được xử lý đúng — kernel chờ hết grace period (mặc định 10s) rồi mới `SIGKILL`.

**Cách chẩn đoán**:
```bash
docker exec <container> ps aux
# Neu PID 1 la /bin/sh -c thay vi ung dung that -> day la nguyen nhan
```

**Cách khắc phục**: dùng `exec` form trong Dockerfile (`CMD ["myapp"]` thay vì `CMD myapp`) để ứng dụng chính trở thành PID 1 trực tiếp, hoặc dùng init tối giản (`tini`, cờ `docker run --init`) để làm PID 1 đúng nghĩa — reap zombie và forward signal đúng chuẩn.

## Sự cố 2: Container bị OOM-killed dù `top` trên host còn dư RAM

**Triệu chứng**: log ứng dụng dừng đột ngột, `docker inspect` cho thấy `OOMKilled: true`, nhưng `free -h` trên host vẫn còn nhiều RAM trống.

**Nguyên nhân gốc**: giới hạn RAM là do **cgroup memory controller** áp cho riêng container đó (`--memory` khi `docker run`), hoàn toàn độc lập với tổng RAM host. Container có thể bị kernel OOM-kill dù host còn dư hàng chục GB, vì cgroup của chính nó đã chạm trần riêng.

**Cách chẩn đoán**:
```bash
dmesg | grep -i "killed process"
# Tim dong: Memory cgroup out of memory: Killed process <pid> (myapp)

cat /sys/fs/cgroup/system.slice/docker-<container_id>.scope/memory.events
# truong "oom_kill" > 0 xac nhan chinh cgroup nay bi OOM
```

**Cách khắc phục**: tăng `--memory` nếu ứng dụng thực sự cần, hoặc tìm memory leak trong ứng dụng bằng cách theo dõi `memory.current` theo thời gian trước khi set lại giới hạn hợp lý.

## Sự cố 3: `docker info` báo `Cgroup Driver: cgroupfs` trên hệ thống dùng `systemd`

**Triệu chứng**: hệ thống thỉnh thoảng có hành vi cgroup không nhất quán (giới hạn RAM/CPU của container bị "reset" bất thường sau khi `systemctl daemon-reload`), đặc biệt gặp khi cùng cài kubelet trên node.

**Nguyên nhân gốc**: có **hai bộ quản lý cgroup** cùng thao tác một cây `/sys/fs/cgroup` — `systemd` (vì nó là init PID 1 và tự quản cgroup cho toàn hệ thống) và Docker (đang cấu hình dùng `cgroupfs` thay vì giao lại cho `systemd`). Khi `systemd` reload, nó có thể ghi đè cấu trúc cgroup mà Docker/runc đã thiết lập.

**Cách khắc phục**: sửa `/etc/docker/daemon.json` thêm `"exec-opts": ["native.cgroupdriver=systemd"]`, `sudo systemctl restart docker`, xác nhận lại `docker info | grep -i "cgroup driver"`. Đây là khuyến nghị bắt buộc cho mọi node production chạy `systemd`.

## Sự cố 4: Image build ra dung lượng lớn bất thường dù Dockerfile ngắn

**Triệu chứng**: `docker images` cho thấy image vài GB dù chỉ vài dòng `RUN apt-get install`.

**Nguyên nhân gốc**: hiểu sai cơ chế layer — mỗi lệnh `RUN` tạo một layer **mới**, và layer là bất biến. Nếu bạn `RUN apt-get update && apt-get install -y build-essential` rồi ở dòng `RUN` kế tiếp mới `apt-get purge build-essential`, layer chứa `build-essential` **vẫn tồn tại vĩnh viễn trong image** (vì OverlayFS chỉ thêm whiteout, không xóa dữ liệu ở layer trước) — image vẫn cõng theo toàn bộ dung lượng đó.

**Cách chẩn đoán**:
```bash
docker history <image>
# Xem tung layer nang bao nhieu, layer nao la thu pham
```

**Cách khắc phục**: gộp các lệnh cài + dọn dẹp trong **cùng một `RUN`** (một layer duy nhất), hoặc dùng multi-stage build để loại bỏ hẳn build tool ra khỏi layer cuối (chi tiết ở Module 10).

## Sự cố 5: `nsenter`/`docker exec` báo "no such process" dù `docker ps` vẫn thấy container Running

**Triệu chứng**: `docker exec -it <container> bash` báo lỗi, nhưng `docker ps` vẫn liệt kê container ở trạng thái `Up`.

**Nguyên nhân gốc**: PID chính của container (PID 1 bên trong namespace) đã chết nhưng `containerd-shim` chưa kịp cập nhật trạng thái, hoặc tiến trình host tương ứng đã bị dọn nhưng metadata Docker chưa đồng bộ — thường do race condition khi ứng dụng bên trong tự restart không đúng cách (ví dụ script wrapper `exec`-replace tiến trình sai PID).

**Cách chẩn đoán**:
```bash
docker inspect -f '{{.State.Pid}}' <container>
# Neu tra ve 0 -> Docker cho rang khong co tien trinh chinh nao dang chay

ps -ef | grep containerd-shim
# Kiem tra shim con song khong, va no dang giu container nao
```

**Cách khắc phục**: `docker restart <container>` để buộc tạo lại toàn bộ namespace/tiến trình sạch; nếu lặp lại thường xuyên, kiểm tra ứng dụng có đang tự `fork`/daemonize sai cách bên trong container hay không (ứng dụng container hóa không nên tự daemonize — phải chạy foreground).

## Sự cố 6: Container ping được ra Internet nhưng container khác trên cùng host không ping được nhau

**Triệu chứng**: hai container trên cùng Docker network không thấy nhau qua IP, dù cả hai đều ra Internet bình thường.

**Nguyên nhân gốc**: thường do container thứ hai bị chạy với `--network none` hoặc gắn sai network (mỗi network trong Docker map tới một Linux bridge/namespace riêng — hai container ở hai network khác nhau nằm trong hai network namespace không nối với nhau, dù cùng host). Đây là ứng dụng thực tế của kiến thức Network Namespace đã học — sẽ đào sâu network driver ở Module 12.

**Cách chẩn đoán**:
```bash
docker inspect <container1> -f '{{.NetworkSettings.Networks}}'
docker inspect <container2> -f '{{.NetworkSettings.Networks}}'
# So sanh xem hai container co chung mot network name khong
```

**Cách khắc phục**: đảm bảo cả hai container cùng khai báo chung một user-defined bridge network (`docker network create` + `--network <tên>`), tránh dùng mạng `bridge` mặc định vì nó không hỗ trợ DNS resolution giữa container theo tên.
