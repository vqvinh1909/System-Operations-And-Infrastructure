---
title: "05 - Labs"
module: 8
tags: [docker, sysops-infra, module-08, labs, hands-on]
---

# 05 - Labs: Thực hành Namespaces, Cgroups, OverlayFS

Môi trường yêu cầu: một VM Linux (Ubuntu 22.04+/Debian 11+ khuyến nghị vì có cgroups v2 mặc định), quyền `sudo`, gói `util-linux` (thường có sẵn). Không cần cài Docker cho các lab này — mục tiêu là tự tay dựng lại cơ chế bên dưới Docker để hiểu tận gốc.

Lưu code/config bạn viết ra trong quá trình làm lab vào thư mục `labs/` (xem `labs/README.md`).

## Lab 1 (Basic): Khảo sát namespace hiện có trên hệ thống

**Mục tiêu**: làm quen với `/proc/<pid>/ns/`, `lsns`, và xác nhận mọi tiến trình host thông thường đều nằm chung một bộ namespace gốc.

**Các bước**:

1. Chạy `lsns` và quan sát output — ghi lại có bao nhiêu namespace mỗi loại đang tồn tại trên hệ thống của bạn.
2. Chọn 3 tiến trình bất kỳ đang chạy trên host (ví dụ `sshd`, `cron`, shell hiện tại của bạn), lấy PID bằng `pgrep` hoặc `ps -ef`.
3. Với mỗi tiến trình, chạy `ls -la /proc/<pid>/ns/` và ghi lại inode number của namespace `pid` và `net`.
4. So sánh: 3 tiến trình này có cùng inode `pid:[...]` và `net:[...]` không? Giải thích vì sao (gợi ý: chúng chưa từng bị `unshare` nên đều thuộc namespace gốc do `systemd` PID 1 tạo ra).
5. Cài `stress-ng` hoặc dùng lệnh `yes > /dev/null &` để tạo một tiến trình con, kiểm tra namespace của tiến trình con này — xác nhận nó kế thừa đúng namespace của tiến trình cha.

**Kết quả cần đạt**: giải thích được bằng chính output thật trên máy bạn rằng "không unshare thì mọi tiến trình chia sẻ chung namespace, inode number giống hệt nhau".

## Lab 2 (Basic): Tạo namespace cô lập bằng `unshare`

**Mục tiêu**: tự tay tạo từng loại namespace, quan sát tác động thực tế.

**Các bước**:

1. Mở 2 terminal (terminal A và terminal B) SSH vào cùng VM.
2. Ở terminal A, chạy `sudo unshare --uts bash`, sau đó `hostname lab-container`.
3. Ở terminal B (không unshare), chạy `hostname` — xác nhận hostname KHÔNG đổi.
4. Ở terminal A, chạy `sudo unshare --pid --fork --mount-proc bash`, sau đó `echo $$` và `ps aux`.
5. Ở terminal B, tìm PID thật của tiến trình bash vừa tạo bằng `ps -ef | grep bash`, so sánh với PID = 1 quan sát được ở terminal A.
6. Kết hợp namespace: `sudo unshare --pid --fork --mount --uts --ipc --net --mount-proc bash`, rồi chạy `ip addr` bên trong — xác nhận chỉ còn thấy `lo`, không thấy interface mạng thật của host.

**Kết quả cần đạt**: một bảng ghi lại từng loại namespace đã tạo, lệnh dùng để kiểm chứng, và kết quả quan sát được (trước/sau khi unshare).

## Lab 3 (Intermediate): Dùng `nsenter` để gia nhập namespace như `docker exec`

**Mục tiêu**: hiểu cơ chế `docker exec` thực sự làm gì bên dưới.

**Điều kiện**: cần Docker đã cài trên VM (nếu chưa có, cài Docker Engine theo hướng dẫn chính thức trước khi làm lab này — việc cài đặt chi tiết sẽ học sâu ở Module 09, ở đây chỉ cần Docker chạy được để lấy PID thử nghiệm).

**Các bước**:

1. Chạy một container nền: `docker run -d --name lab-nginx nginx`.
2. Lấy PID thật trên host của tiến trình chính: `PID=$(docker inspect -f '{{.State.Pid}}' lab-nginx)`.
3. Dùng `nsenter --target $PID --all bash` để chui vào toàn bộ namespace của container mà KHÔNG dùng `docker exec`.
4. Bên trong, chạy `hostname`, `ip addr`, `ps aux`, `cat /etc/os-release` — xác nhận bạn đang thấy đúng môi trường của container.
5. So sánh thời gian phản hồi và kết quả giữa `nsenter` thủ công và `docker exec -it lab-nginx bash` — chúng phải cho kết quả giống hệt nhau.
6. Dọn dẹp: `docker rm -f lab-nginx`.

**Kết quả cần đạt**: chứng minh bằng thực nghiệm rằng `docker exec` không có "phép thuật" gì đặc biệt — nó chỉ gọi `setns()` đúng các namespace mà container sở hữu.

## Lab 4 (Intermediate): Giới hạn tài nguyên bằng cgroups v2 thủ công

**Mục tiêu**: tự tay tạo cgroup, áp giới hạn CPU/RAM, quan sát kernel enforce giới hạn.

**Các bước**:

1. Xác nhận hệ thống dùng cgroups v2: `stat -fc %T /sys/fs/cgroup/` phải trả về `cgroup2fs`.
2. Tạo cgroup con: `sudo mkdir /sys/fs/cgroup/lab-cgroup`.
3. Giới hạn RAM 50MB: `echo "50M" | sudo tee /sys/fs/cgroup/lab-cgroup/memory.max`.
4. Đưa shell hiện tại vào cgroup: `echo $$ | sudo tee /sys/fs/cgroup/lab-cgroup/cgroup.procs`.
5. Trong đúng shell đó, chạy một chương trình cố tình cấp phát RAM vượt giới hạn (ví dụ dùng `stress-ng --vm 1 --vm-bytes 200M --timeout 10s` hoặc một script Python cấp phát mảng lớn) và quan sát tiến trình bị kernel OOM-kill.
6. Kiểm tra `dmesg | tail -30` để tìm dòng log `Memory cgroup out of memory: Killed process ...` — đây chính là bằng chứng cgroup đã chặn đúng như cấu hình.
7. Lặp lại tương tự với giới hạn CPU: set `cpu.max` = `20000 100000` (giới hạn 20% một core), chạy `stress-ng --cpu 1 --timeout 15s`, quan sát bằng `top` ở terminal khác để thấy %CPU bị ép trần quanh 20%.
8. Kiểm tra `cat /sys/fs/cgroup/lab-cgroup/cpu.stat`, ghi lại giá trị `nr_throttled` và `throttled_usec`.
9. Dọn dẹp: đưa tiến trình trở lại cgroup gốc rồi `sudo rmdir /sys/fs/cgroup/lab-cgroup`.

**Kết quả cần đạt**: log `dmesg` chứng minh OOM-kill do cgroup, và số liệu `cpu.stat` chứng minh CPU throttling — hai loại sự cố này chính là những gì bạn sẽ chẩn đoán thường xuyên khi vận hành container ở production.

## Lab 5 (Advanced / Mini Project): Tự xây "mini container runtime" bằng `unshare` + `chroot` + cgroups

**Mục tiêu**: kết hợp toàn bộ kiến thức của module — namespace, cgroup, root filesystem riêng — để tự tay dựng một tiến trình cô lập giống hệt về bản chất với những gì `runc` làm khi Docker chạy `docker run`.

**Chuẩn bị root filesystem tối giản bằng BusyBox**:

1. Tạo thư mục gốc cho "mini container": `sudo mkdir -p /opt/mini-container/rootfs`.
2. Cài `busybox-static` (hoặc tương đương) và tạo cấu trúc thư mục cơ bản (`/bin`, `/proc`, `/sys`, `/dev`, `/etc`) bên trong `rootfs`, copy binary `busybox` tĩnh vào `rootfs/bin/busybox`, rồi tạo các symlink lệnh cơ bản (`ls`, `sh`, `ps`, `mount`, `hostname`...) trỏ về `busybox` bằng `busybox --install`.

**Dựng cô lập từng bước**:

3. Tạo cgroup riêng cho mini container, giới hạn RAM 100MB và CPU 50%: `sudo mkdir /sys/fs/cgroup/mini-container` rồi set `memory.max` và `cpu.max` như Lab 4.
4. Chạy lệnh tổng hợp:
   ```bash
   sudo unshare --pid --fork --mount --uts --ipc --net --mount-proc \
     chroot /opt/mini-container/rootfs /bin/sh
   ```
5. Ngay sau khi vào được shell bên trong, tự đưa PID hiện tại vào cgroup đã tạo ở bước 3 (thực hiện từ một terminal khác vì PID namespace mới khiến bạn cần biết PID thật trên host).
6. Bên trong "mini container", thử: `hostname mini01`, `ps aux` (chỉ thấy tiến trình của chính nó nhờ `--mount-proc`), `ls /` (chỉ thấy filesystem BusyBox tối giản, không thấy filesystem thật của host), `ip addr` (không có interface mạng ra ngoài vì chưa cấu hình `veth`).
7. Từ host, quan sát: `ps -ef | grep sh` để thấy PID thật, và `cat /sys/fs/cgroup/mini-container/memory.current` để xác nhận tiến trình đang bị theo dõi RAM đúng cgroup đã set.

**Mở rộng (tùy chọn, nâng cao hơn)**: tạo một cặp `veth`, đưa một đầu vào network namespace của mini container, nối đầu còn lại vào một bridge trên host, cấp IP nội bộ cho mini container để nó có thể ping ra ngoài — đây chính xác là việc Docker làm khi tạo container gắn vào `docker0`.

**Kết quả cần đạt**: một script bash (lưu vào `labs/mini-container.sh`) tự động hóa toàn bộ các bước trên, kèm ghi chú (comment trong script) giải thích mỗi bước tương ứng với cơ chế nào (namespace loại gì, cgroup controller nào, OverlayFS/chroot đóng vai trò gì). Đây là bài tập tổng hợp chuẩn bị trực tiếp cho việc hiểu sâu Module 09 khi bạn bắt đầu dùng `docker run` — bạn sẽ nhận ra mọi cờ (`--network`, `--memory`, `--cpus`, `--hostname`...) của lệnh đó ánh xạ trực tiếp tới chính những gì bạn vừa tự tay làm ở lab này.
