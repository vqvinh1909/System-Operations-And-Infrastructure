---
title: "04 - Commands"
module: 8
tags: [docker, sysops-infra, module-08, commands, unshare, nsenter, cgroups]
---

# 04 - Lệnh và Ví dụ Cấu hình: Namespaces, Cgroups, OverlayFS

Toàn bộ lệnh trong file này KHÔNG cần Docker — dùng công cụ có sẵn trong gói `util-linux` (`unshare`, `nsenter`, `lsns`) để bạn thấy trực tiếp kernel làm gì, trước khi trừu tượng hóa qua Docker CLI ở Module 09.

## 1. Khảo sát namespace hiện có trên hệ thống

```bash
# Liệt kê toàn bộ namespace đang tồn tại trên host
lsns

# Lọc theo loại namespace, ví dụ chỉ xem PID namespace
lsns -t pid

# Xem namespace của chính tiến trình shell hiện tại
ls -la /proc/$$/ns/
```

Output mẫu của `ls -la /proc/$$/ns/`:
```text
lrwxrwxrwx 1 root root 0 Jul 12 10:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jul 12 10:00 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Jul 12 10:00 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Jul 12 10:00 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Jul 12 10:00 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jul 12 10:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jul 12 10:00 uts -> 'uts:[4026531838]'
```

Con số trong `[...]` là inode number của namespace. Hai tiến trình có cùng số này ở cùng loại namespace nghĩa là chúng **chia sẻ chung namespace đó**.

## 2. Dùng `unshare` để tạo namespace mới

Cú pháp tổng quát: `unshare [OPTIONS] -- lenh_can_chay`

### 2.1 Tạo UTS namespace riêng (đổi hostname không ảnh hưởng host)

```bash
sudo unshare --uts bash
# Bên trong shell mới:
hostname sandbox-container
hostname
# -> sandbox-container

# Mở một terminal KHÁC (không unshare), kiểm tra hostname host
hostname
# -> hostname that cua host, khong doi
```

### 2.2 Tạo PID namespace riêng — chứng minh PID 1 bên trong khác PID thật trên host

```bash
sudo unshare --pid --fork --mount-proc bash
# Bên trong:
echo $$
# -> 1  (PID 1 BEN TRONG namespace nay)
ps aux
# -> chi thay tien trinh cua chinh no, khong thay tien trinh host

# Tren mot terminal khac (host), tim PID that cua tien trinh do:
ps -ef | grep bash
# -> se thay mot PID host binh thuong, vi du 45231, khac han so 1 nhin thay ben trong
```

Cờ `--mount-proc` bắt buộc khi tạo PID namespace mới, vì nó remount `/proc` để lệnh `ps` bên trong đọc đúng thông tin PID namespace mới thay vì đọc nhầm `/proc` của host.

### 2.3 Kết hợp nhiều namespace cùng lúc (gần giống những gì Docker làm)

```bash
sudo unshare --pid --fork --mount --uts --ipc --net --mount-proc bash
# Bay gio ben trong da co: PID namespace rieng, mount namespace rieng,
# hostname rieng, IPC rieng, va network namespace rieng (khong co interface nao ngoai lo)

ip addr
# -> chi thay "lo" (loopback), khong thay eth0/enp... cua host
```

Nhận xét: đến bước này, tiến trình bash bên trong đã bị cô lập gần giống một container thật — chỉ còn thiếu cgroup (giới hạn tài nguyên) và một root filesystem riêng (thường dùng `chroot`/`pivot_root` — sẽ thực hành ở `05-Labs.md`).

### 2.4 User namespace — map UID root bên trong thành UID thường trên host

```bash
unshare --user --map-root-user bash
# Khong can sudo cho lenh nay (user namespace la namespace duy nhat
# mot tien trinh khong dac quyen cung tao duoc)

whoami
# -> root (BEN TRONG namespace)
id
# -> uid=0(root) gid=0(root)

# Tren host, tim tien trinh that va xem UID that:
# UID that tren host van la UID cua user goc chay lenh unshare,
# KHONG phai root that su.
```

## 3. Dùng `nsenter` để "chui vào" namespace đã tồn tại

`nsenter` là cơ chế nền tảng mà `docker exec` sử dụng để chạy lệnh mới bên trong một container đang chạy.

```bash
# Buoc 1: lay PID cua tien trinh muc tieu (vi du PID cua mot container process)
TARGET_PID=1651

# Buoc 2: gia nhap dung cac namespace cua tien trinh do va chay bash
sudo nsenter --target $TARGET_PID --pid --net --mount --uts --ipc bash

# Hoac gia nhap TOAN BO namespace cua tien trinh muc tieu:
sudo nsenter --target $TARGET_PID --all bash
```

Trong thực tế với Docker, bạn có thể tự tay làm điều `docker exec` làm bên dưới:

```bash
# Lay PID that tren host cua tien trinh chinh trong container
CID=$(docker ps -q --filter "name=my-nginx")
PID=$(docker inspect -f '{{.State.Pid}}' $CID)

# Chui vao dung namespace cua container do ma khong can Docker CLI
sudo nsenter --target $PID --all bash
```

## 4. Cgroups v2 — kiểm tra và cấu hình giới hạn tài nguyên tay

### 4.1 Kiểm tra hệ thống đang dùng cgroups version nào

```bash
mount | grep cgroup
# Neu thay dong "cgroup2 on /sys/fs/cgroup type cgroup2" -> he thong dung cgroups v2 (unified)
# Neu thay nhieu dong "cgroup on /sys/fs/cgroup/xxx type cgroup" rieng le -> cgroups v1

stat -fc %T /sys/fs/cgroup/
# -> "cgroup2fs" nghia la v2, "tmpfs" nghia la v1 (hybrid mode)
```

### 4.2 Kiểm tra cgroup driver Docker đang dùng

```bash
docker info | grep -i cgroup
# Cgroup Driver: systemd     <- khuyen nghi cho production
# Cgroup Version: 2
```

Nếu output là `Cgroup Driver: cgroupfs` trên hệ thống dùng `systemd` làm init, cần cấu hình lại (xem `06-Troubleshooting.md`).

### 4.3 Ví dụ cấu hình `daemon.json` để ép Docker dùng cgroup driver `systemd`

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Đường dẫn file: `/etc/docker/daemon.json`. Sau khi sửa, bắt buộc restart daemon:

```bash
sudo systemctl restart docker
docker info | grep -i "cgroup driver"
```

### 4.4 Tự tay tạo một cgroup và giới hạn CPU/RAM (không qua Docker)

```bash
# Tao mot cgroup con moi trong cgroups v2
sudo mkdir /sys/fs/cgroup/demo-group

# Gioi han RAM toi da 100MB cho cgroup nay
echo "100M" | sudo tee /sys/fs/cgroup/demo-group/memory.max

# Gioi han CPU: 50% cua 1 core (quota 50000 tren period 100000 microsecond)
echo "50000 100000" | sudo tee /sys/fs/cgroup/demo-group/cpu.max

# Dua tien trinh shell hien tai vao cgroup nay
echo $$ | sudo tee /sys/fs/cgroup/demo-group/cgroup.procs

# Kiem tra cac tien trinh dang thuoc cgroup nay
cat /sys/fs/cgroup/demo-group/cgroup.procs

# Chay mot lenh ton nhieu CPU de kiem chung gioi han (vi du stress-ng neu co cai)
stress-ng --cpu 1 --timeout 20s
# Dung "top" tren mot terminal khac de quan sat %CPU bi ep tran o muc ~50%
```

### 4.5 Xem thống kê tài nguyên thực tế của một cgroup

```bash
cat /sys/fs/cgroup/demo-group/memory.current   # RAM dang dung
cat /sys/fs/cgroup/demo-group/memory.max       # gioi han RAM da dat
cat /sys/fs/cgroup/demo-group/cpu.stat         # thoi gian CPU da dung, so lan bi throttle
cat /sys/fs/cgroup/demo-group/pids.current     # so luong tien trinh hien tai trong cgroup
```

`cpu.stat` có trường `nr_throttled` và `throttled_usec` — đây chính xác là những con số Docker/Kubernetes đọc để báo cáo "container bị CPU throttling", một triệu chứng hiệu năng rất hay gặp khi set `--cpus` quá thấp cho ứng dụng cần burst CPU.

## 5. OverlayFS — tự tay mount thử để hiểu cơ chế

```bash
# Chuan bi thu muc cho tung thanh phan
mkdir -p /tmp/overlay-demo/{lower,upper,merged,work}

# Tao du lieu mau trong lower (dai dien cho "image layer")
echo "noi dung goc tu image" > /tmp/overlay-demo/lower/file.txt

# Mount overlay
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/overlay-demo/lower,upperdir=/tmp/overlay-demo/upper,workdir=/tmp/overlay-demo/work \
  /tmp/overlay-demo/merged

# Kiem tra view hop nhat
cat /tmp/overlay-demo/merged/file.txt
# -> "noi dung goc tu image"

# Sua file qua view merged (mo phong container ghi du lieu)
echo "da bi sua boi container" >> /tmp/overlay-demo/merged/file.txt

# Kiem chung: file GOC trong lower KHONG doi (bat bien)
cat /tmp/overlay-demo/lower/file.txt
# -> van chi co "noi dung goc tu image"

# File da duoc copy-up sang upper va sua o do
cat /tmp/overlay-demo/upper/file.txt
# -> "noi dung goc tu image" + dong moi them vao

# Don dep sau khi thu nghiem
sudo umount /tmp/overlay-demo/merged
rm -rf /tmp/overlay-demo
```

## 6. Lệnh tra cứu nhanh (cheat sheet)

| Lệnh | Mục đích |
|---|---|
| `lsns` | Liệt kê toàn bộ namespace trên host |
| `lsns -t <type>` | Lọc theo loại namespace (pid, net, mnt, uts, ipc, user, cgroup) |
| `ls -la /proc/<pid>/ns/` | Xem namespace của một tiến trình cụ thể |
| `unshare [--pid\|--net\|--mnt\|--uts\|--ipc\|--user] --fork <cmd>` | Tạo namespace mới và chạy lệnh trong đó |
| `nsenter --target <pid> --all <cmd>` | Gia nhập toàn bộ namespace của tiến trình đích |
| `docker inspect -f '{{.State.Pid}}' <container>` | Lấy PID thật trên host của tiến trình chính trong container |
| `mount \| grep cgroup` | Kiểm tra cgroups v1 hay v2 đang active |
| `docker info \| grep -i cgroup` | Kiểm tra cgroup driver và version Docker đang dùng |
| `cat /sys/fs/cgroup/<group>/memory.current` | RAM hiện tại của một cgroup |
| `cat /sys/fs/cgroup/<group>/cpu.stat` | Thống kê CPU, số lần bị throttle |
| `mount -t overlay overlay -o lowerdir=...,upperdir=...,workdir=... <merged>` | Mount thủ công một OverlayFS |
