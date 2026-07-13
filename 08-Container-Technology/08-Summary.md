---
title: "08 - Summary"
module: 8
tags: [docker, sysops-infra, module-08, summary]
---

# 08 - Summary: Container Technology

## Những gì đã học

- **VM vs Container**: VM ảo hóa phần cứng qua Hypervisor, mỗi VM có kernel riêng — cách ly mạnh, nặng, chậm boot. Container chia sẻ chung kernel Host OS, cô lập bằng namespace + giới hạn bằng cgroup — nhẹ, khởi động gần tức thì, cách ly yếu hơn.
- **OCI**: chuẩn chung của ngành gồm Runtime Spec (`runc` chạy container), Image Spec (cấu trúc image/layer), Distribution Spec (API registry) — đảm bảo tương thích giữa các vendor.
- **Namespaces**: PID, Network, Mount, UTS, IPC, User — mỗi loại cô lập một "view" tài nguyên hệ thống khác nhau. `unshare` tạo namespace mới, `nsenter`/`setns()` gia nhập namespace đã tồn tại (chính là cơ chế `docker exec`).
- **Cgroups v1 vs v2**: v1 nhiều cây phân cấp riêng theo controller, v2 một cây thống nhất (mặc định trên distro hiện đại, systemd ≥ 247). Driver `systemd` là khuyến nghị production để tránh xung đột với init system.
- **OverlayFS**: `lowerdir` (image layer, read-only) + `upperdir` (container layer, read-write) hợp nhất thành `merged`, dùng cơ chế copy-on-write (copy-up khi ghi, whiteout file khi xóa) — image layer bất biến, container khởi động không cần copy dữ liệu.
- **Image Layer**: mỗi lệnh Dockerfile thay đổi filesystem tạo một layer mới, định danh bằng SHA-256 digest (content-addressable) — cho phép cache build và chia sẻ layer giữa nhiều image.

## Checklist tự đánh giá

- [ ] Vẽ được sơ đồ VM vs Container từ trí nhớ, giải thích đúng vị trí Hypervisor/Guest OS bị thay bằng Container Runtime.
- [ ] Liệt kê đủ 6 namespace và giải thích đúng mỗi loại cô lập cái gì.
- [ ] Phân biệt được cgroups v1/v2 và giải thích lý do chọn driver `systemd`.
- [ ] Giải thích được copy-on-write của OverlayFS bằng ví dụ đọc/ghi/xóa file cụ thể.
- [ ] Tự chạy được lab `unshare` + `chroot` tạo mini container thủ công, không cần nhìn tài liệu.
- [ ] Đọc được `dmesg`/`cpu.stat`/`memory.current` để chẩn đoán OOM-kill và CPU throttling.

## Liên kết tới Module tiếp theo

Toàn bộ 4 cơ chế kernel (namespace, cgroup, OverlayFS, image layer) chính là "phần cứng lý thuyết" mà [[../09-Docker-Engine/README|Module 09 - Docker Engine]] sẽ đóng gói thành trải nghiệm quen thuộc: `dockerd` nhận lệnh từ CLI, giao cho `containerd` quản lý vòng đời, `containerd-shim` giữ container sống độc lập với `dockerd`, và `runc` là bên thực sự gọi `clone()`/`unshare()`/mount OverlayFS mà bạn vừa tự tay làm thủ công ở module này. Mọi cờ lệnh `docker run --network`, `--memory`, `--cpus`, `--hostname`... từ Module 09 trở đi đều ánh xạ trực tiếp tới một namespace hoặc cgroup controller bạn đã học ở đây.

> [!tip] Ghi nhớ cốt lõi
> Container không phải "VM nhẹ" — nó là tiến trình Linux bình thường được kernel cô lập bằng namespace và giới hạn bằng cgroup, chạy trên root filesystem dựng từ các layer OverlayFS bất biến. Hiểu đúng bản chất này là điều phân biệt một Sysadmin thao tác Docker theo bản năng với một người thực sự troubleshoot được production.
