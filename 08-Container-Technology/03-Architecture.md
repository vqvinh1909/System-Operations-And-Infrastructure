---
title: "03 - Architecture"
module: 8
tags: [docker, sysops-infra, module-08, architecture, diagram]
---

# 03 - Kiến trúc: Sơ đồ VM vs Container, Process, OverlayFS

## 1. Sơ đồ so sánh: Virtual Machine Architecture

VM ảo hóa toàn bộ phần cứng qua Hypervisor. Mỗi VM cõng theo một Guest OS đầy đủ, có kernel riêng — đây là lý do VM nặng và khởi động chậm.

```text
+----------------------------------------------------------------+
|                  VIRTUAL MACHINE ARCHITECTURE                  |
|----------------------------------------------------------------|
|            App A            App B            App C             |
|----------------------------------------------------------------|
|          Bin/Lib A        Bin/Lib B        Bin/Lib C           |
|----------------------------------------------------------------|
|         Guest OS #1      Guest OS #2      Guest OS #3          |
|        (kernel rieng)   (kernel rieng)   (kernel rieng)        |
|----------------------------------------------------------------|
|                     HYPERVISOR (Type 1/2)                      |
|            VMware ESXi / KVM / Hyper-V / VirtualBox            |
|----------------------------------------------------------------|
|        HOST OPERATING SYSTEM (neu la Type 2 Hypervisor)        |
|----------------------------------------------------------------|
|            PHYSICAL HARDWARE (CPU, RAM, Disk, NIC)             |
+----------------------------------------------------------------+
```

Đọc từ dưới lên: phần cứng vật lý bị Hypervisor "cắt lát" thành phần cứng ảo, mỗi lát được cấp cho một Guest OS boot đầy đủ như một máy thật độc lập. App A, App B, App C hoàn toàn không biết sự tồn tại của nhau, và mỗi Guest OS quản lý driver, tiến trình, kernel của riêng nó.

## 2. Sơ đồ so sánh: Container Architecture

Container không có tầng Hypervisor, không có Guest OS riêng. Container Runtime tạo cô lập bằng namespace/cgroup ngay trên kernel của Host OS — mọi container **chia sẻ chung một kernel**.

```text
+----------------------------------------------------------------+
|                     CONTAINER ARCHITECTURE                     |
|----------------------------------------------------------------|
|        Container A       Container B       Container C         |
|----------------------------------------------------------------|
|      App + Bin/Lib A   App + Bin/Lib B   App + Bin/Lib C       |
|----------------------------------------------------------------|
|       (namespace + cgroup rieng, KHONG co kernel rieng)        |
|----------------------------------------------------------------|
|                       CONTAINER RUNTIME                        |
|        dockerd -> containerd -> containerd-shim -> runc        |
|----------------------------------------------------------------|
|                     HOST OPERATING SYSTEM                      |
|           Linux Kernel DUNG CHUNG cho moi container            |
|----------------------------------------------------------------|
|            PHYSICAL HARDWARE (CPU, RAM, Disk, NIC)             |
+----------------------------------------------------------------+
```

So sánh trực tiếp với sơ đồ VM ở trên: vị trí "Guest OS + Hypervisor" bên VM được thay bằng "Container Runtime" mỏng hơn hẳn — không có bước ảo hóa phần cứng, không có bước boot một kernel mới. Container A/B/C chỉ là các tiến trình trên Host OS được kernel cô lập bằng namespace (không nhìn thấy nhau) và giới hạn bằng cgroup (không giành hết tài nguyên của nhau). Đây là lý do cốt lõi khiến container khởi động nhanh hơn VM hàng trăm lần và mật độ triển khai trên một host cao hơn hẳn.

**Chuỗi tiến trình chạy một container** (khớp với Container Runtime ở sơ đồ trên): `dockerd` (daemon nhận lệnh từ Docker CLI) giao việc cho `containerd` (quản lý vòng đời container ở tầng thấp hơn), `containerd` sinh ra một tiến trình `containerd-shim` riêng cho mỗi container (shim là tiến trình cha thực sự của container, giúp container sống sót ngay cả khi `dockerd` restart), và `containerd-shim` gọi `runc` — chính `runc` là bên thực sự gọi các system call `clone()`/`unshare()` tạo namespace, gắn cgroup, dựng root filesystem, rồi `exec` tiến trình ứng dụng vào bên trong.

## 3. Sơ đồ: PID Namespace lồng nhau trên Host

Namespace không "cắt đứt" quan hệ tiến trình — nó chỉ giới hạn *tầm nhìn*. Trên Host, tiến trình container vẫn có PID thật (host nhìn thấy toàn bộ), nhưng bên trong PID namespace riêng, chính tiến trình đó lại mang PID 1.

```text
+--------------------------------------------------------------------+
| HOST - PID NAMESPACE GOC (PID NS level 0)                          |
|                                                                    |
|   systemd (PID 1 that, toan bo tien trinh host nhin thay)          |
|   |-- dockerd (PID 1500 tren host)                                 |
|   |-- containerd-shim (PID 1650 tren host)                         |
|   |     `-- container process (PID 1651 tren host)                 |
|   |            = PID 1 BEN TRONG PID namespace cua container       |
|   |-- sshd, cron, nginx... (cac tien trinh host khac)              |
+--------------------------------------------------------------------+
```

Đây là điểm hay gây nhầm lẫn với người mới: chạy `ps aux` trên **host** sẽ thấy tiến trình container với một PID "bình thường" (ví dụ 1651). Nhưng chạy `ps aux` **bên trong container** (qua `docker exec` hoặc `nsenter`) chỉ thấy PID 1 cho đúng tiến trình đó, vì bên trong PID namespace riêng, kernel đánh số lại từ đầu. Hai con số PID khác nhau cùng trỏ tới một tiến trình vật lý duy nhất trong kernel — chỉ là "nhìn" từ hai namespace khác nhau.

## 4. Sơ đồ: OverlayFS — xếp lớp Image và Container Layer

`lowerdir` là các layer chỉ đọc của image (xếp từ dưới lên theo thứ tự Dockerfile), `upperdir` là layer ghi riêng của container, `merged` là view cuối cùng container thực sự dùng, `workdir` là vùng làm việc nội bộ bắt buộc của OverlayFS.

```text
+------------------------------------------------------------------+
| merged/  (view tong hop ma container thuc su nhin thay va dung)  |
+------------------------------------------------------------------+
        ^
        | hop nhat (union mount - doc)
+------------------------------------------------------------------+
|   upperdir/  (RW - layer ghi cua container, luu thay doi/xoa)    |
+------------------------------------------------------------------+
        ^
        | copy-up khi ghi/xoa (copy-on-write)
+------------------------------------------------------------------+
|         lowerdir layer 3 (RO) - image layer: COPY app.py         |
+------------------------------------------------------------------+
+------------------------------------------------------------------+
|    lowerdir layer 2 (RO) - image layer: RUN pip install flask    |
+------------------------------------------------------------------+
+------------------------------------------------------------------+
|    lowerdir layer 1 (RO) - image layer: FROM python:3.12-slim    |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|workdir/ (vung lam viec noi bo overlay, khong truy cap truc tiep) |
+------------------------------------------------------------------+
```

Đọc sơ đồ: ba `lowerdir` layer (1, 2, 3) chính là ba layer bất biến sinh ra từ ba dòng lệnh trong Dockerfile (`FROM`, `RUN`, `COPY`) — xếp chồng theo đúng thứ tự build. Khi container khởi động, kernel gắn thêm một `upperdir` trống lên trên cùng để chứa mọi thay đổi khi container chạy (container layer), và trình bày toàn bộ tổ hợp đó thành một cây thư mục duy nhất tại `merged` — đây chính là root filesystem (`/`) mà tiến trình bên trong container nhìn thấy qua `mnt` namespace. `workdir` không xuất hiện trong luồng đọc, chỉ được OverlayFS dùng nội bộ để đảm bảo thao tác ghi diễn ra an toàn (atomic).

Nếu bạn `docker commit` một container đang chạy, Docker đóng băng nội dung hiện tại của `upperdir` thành một layer chỉ đọc mới — đây là cách một container "biến" thành layer mới của image, khớp hoàn toàn với lý thuyết Image Layer ở `02-Theory.md`.
