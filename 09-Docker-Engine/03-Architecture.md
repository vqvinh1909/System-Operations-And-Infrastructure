---
title: "03 - Architecture"
module: 9
tags: [docker, sysops-infra, module-09, architecture, diagram]
---

# 03 - Sơ đồ kiến trúc

Ba sơ đồ dưới đây minh họa: (1) luồng gọi đầy đủ từ CLI tới tiến trình container, (2) vòng đời trạng thái container, (3) chuỗi rủi ro bảo mật khi user thuộc nhóm `docker`. Toàn bộ sơ đồ vẽ bằng ASCII, đã được đo và xác nhận từng khung hình chữ nhật có border trên/dưới/nội dung khớp độ dài bằng script Python (xem ghi chú Self-Review trong [[README]]).

## Sơ đồ 1: Luồng gọi từ Docker CLI tới tiến trình container

Đây là sơ đồ quan trọng nhất của module — hãy đảm bảo bạn có thể tự vẽ lại (kể cả trên giấy khi phỏng vấn) và giải thích từng mũi tên.

```
+-----------------------------+
| Docker CLI (docker run ...) |
+-----------------------------+

   |
   .  REST API qua Unix socket /var/run/docker.sock
   v

+----------------------------------+
| dockerd (Docker Daemon)          |
| - Quan ly image, network, volume |
| - Nhan request qua REST API      |
+----------------------------------+

   |
   .  gRPC API (containerd.sock)
   v

+-------------------------------------------+
| containerd                                |
| - Quan ly vong doi container              |
| - Doc lap voi dockerd (co the chay rieng) |
+-------------------------------------------+

   |
   .  fork/exec containerd-shim-runc-v2
   v

+-----------------------------------+
| containerd-shim (shim process)    |
| - Giu container khong bi "mo coi" |
| - Bao cao exit code ve containerd |
+-----------------------------------+

   |
   .  goi runc create/start
   v

+-----------------------------------------+
| runc (OCI Runtime)                      |
| - Tao namespace, cgroup                 |
| - Goi execve() de tao container process |
+-----------------------------------------+

   |
   .  execve() -> tien trinh thuc thi
   v

+------------------------------------+
| Tien trinh trong container (PID 1) |
+------------------------------------+
```

**Đọc sơ đồ này như thế nào:**

- Hai tầng trên cùng (`Docker CLI` → `dockerd`) giao tiếp qua **REST API** trên **Unix socket** — đây là ranh giới client-server đã phân tích ở [[02-Theory]] mục 1.
- Tầng `dockerd` → `containerd` giao tiếp qua **gRPC**, không phải REST — hai giao thức khác nhau cho hai mục đích khác nhau (REST hướng người dùng cuối/công cụ bên ngoài, gRPC hướng nội bộ hệ thống, hiệu năng cao hơn).
- `containerd` → `containerd-shim`: đây là bước **fork/exec**, tạo ra một tiến trình hệ điều hành mới, độc lập với cả `containerd` lẫn `dockerd` — chính bước này là nền tảng giúp container sống sót qua việc restart daemon.
- `containerd-shim` → `runc`: shim gọi `runc` như một tiến trình con ngắn hạn để thực sự tạo container; `runc` tạo xong thì thoát, shim ở lại làm cha của tiến trình container.
- `runc` → tiến trình container: bước cuối cùng là `execve()` — lệnh hệ thống nạp chương trình entrypoint chạy bên trong các namespace vừa được tạo (PID namespace riêng nên tiến trình này có PID = 1 nhìn từ bên trong container, dù trên host nó có một PID bình thường khác).

## Sơ đồ 2: Vòng đời Container (Container Lifecycle State Diagram)

```
Vong doi Container (Container Lifecycle)

+-----------------+
| CREATED         |
| (docker create) |
+-----------------+

   |
   .  docker start
   v

+----------------+
| RUNNING        |
| (docker start) |
+----------------+

   |
   .  docker pause (freeze bang cgroup freezer)
   v

+----------------+
| PAUSED         |
| (docker pause) |
+----------------+

   |
   .  docker unpause -> quay lai trang thai RUNNING o tren

Ghi chu: ca RUNNING va PAUSED deu co the chuyen thang sang EXITED:
  - docker stop  : gui SIGTERM, cho grace period (mac dinh 10s),
                   het gio ma tien trinh chua thoat thi gui SIGKILL
  - docker kill  : gui SIGKILL (hoac signal chi dinh) ngay lap tuc

+-------------------------------+
| EXITED                        |
| (container dung, con ton tai) |
+-------------------------------+

   |
   .  docker rm
   v

+-----------------------------+
| REMOVED                     |
| (docker rm - xoa hoan toan) |
+-----------------------------+
```

**Các điểm cần khắc sâu:**

- `CREATED` là trạng thái "đã cấp phát nhưng chưa chạy" — container có ID, có filesystem, có network config, nhưng tiến trình entrypoint chưa được `execve()`.
- `PAUSED` không đi qua tín hiệu (signal) như `stop`/`kill` — nó dùng cơ chế cgroup freezer ở tầng kernel, hoàn toàn trong suốt với tiến trình bên trong.
- `EXITED` là trạng thái container **vẫn còn tồn tại** trên hệ thống (writable layer, log, exit code vẫn giữ nguyên, `docker inspect` vẫn đọc được) — chỉ khi `docker rm` container mới thực sự biến mất. Đây là lý do `docker ps -a` (có cờ `-a`) vẫn liệt kê được container đã exited, trong khi `docker ps` (không cờ) chỉ hiện container đang `running`.
- Không có mũi tên trực tiếp `CREATED → EXITED` trong sơ đồ trên vì thực tế container ở trạng thái `created` chưa có tiến trình để gửi tín hiệu — nhưng `docker rm` **vẫn xóa được** container đang ở `created` (không cần qua `running`/`exited`), chỉ là không có "signal" nào liên quan vì chưa từng có tiến trình chạy.

## Sơ đồ 3: Rủi ro bảo mật — nhóm `docker` tương đương quyền root

Sơ đồ này minh họa lại phần phân tích bảo mật ở [[02-Theory]] mục 4, giúp hình dung trực quan tại sao "chỉ thêm user vào một nhóm" lại có hậu quả lớn tới vậy.

```
Rui ro bao mat: thanh vien nhom docker = root-equivalent

+--------------------------+
| User trong nhom 'docker' |
| (vd: vinh)               |
+--------------------------+

   |
   .  la thanh vien nhom docker nen ket noi duoc
   .  socket ma khong can sudo
   v

+------------------------------------+
| /var/run/docker.sock               |
| owner=root, group=docker, mode=660 |
+------------------------------------+

   |
   .  moi lenh docker deu duoc dockerd thuc thi
   v

+-----------------------------+
| dockerd chay voi quyen root |
+-----------------------------+

   |
   .  vd: docker run -v /:/host -it alpine chroot /host
   v

+---------------------------------------+
| Toan quyen root tren host             |
| (mount host filesystem, --privileged, |
| sua iptables, doc /etc/shadow...)     |
+---------------------------------------+
```

**Ghi nhớ cho công việc thực tế**: mỗi lần bạn chuẩn bị chạy `usermod -aG docker <username>`, hãy tự hỏi câu hỏi tương đương với việc chuẩn bị chạy `usermod -aG sudo <username>` (hoặc thêm dòng NOPASSWD vào sudoers). Về mặt rủi ro bảo mật, hai hành động này gần như tương đương nhau trên server đó.

---

Tiếp theo: [[04-Commands]] để xem cú pháp lệnh cụ thể áp dụng các khái niệm vừa học.
