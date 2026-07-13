---
title: "07 - Interview"
module: 8
tags: [docker, sysops-infra, module-08, interview]
---

# 07 - Interview: Container Technology

## Junior

**1. Container khác gì Virtual Machine?**
Container chia sẻ chung kernel của Host OS, cô lập bằng namespace và giới hạn tài nguyên bằng cgroup — không ảo hóa phần cứng, không có Guest OS riêng. VM ảo hóa toàn bộ phần cứng qua Hypervisor, mỗi VM có kernel riêng, boot đầy đủ như máy thật. Hệ quả: container khởi động nhanh hơn (mili-giây so với chục giây), nhẹ hơn (MB so với GB), nhưng cách ly yếu hơn VM.

**2. Namespace là gì? Kể tên các loại namespace bạn biết.**
Namespace là tính năng kernel Linux giới hạn một nhóm tiến trình chỉ nhìn thấy một "view" riêng của một loại tài nguyên hệ thống. 6 loại cơ bản dùng cho container: PID (cây tiến trình), Network (network stack), Mount (mount point), UTS (hostname), IPC (message queue/shared memory), User (UID/GID).

**3. Cgroup dùng để làm gì?**
Control Groups giới hạn, đo lường và ưu tiên việc sử dụng tài nguyên (CPU, RAM, I/O, số PID) cho một nhóm tiến trình. Nếu namespace trả lời "container nhìn thấy gì", cgroup trả lời "container được dùng bao nhiêu tài nguyên vật lý".

**4. Image layer là gì?**
Mỗi lệnh `RUN`/`COPY`/`ADD` trong Dockerfile tạo ra một layer filesystem mới (bất biến, chỉ đọc), xếp chồng lên layer trước. Cơ chế này cho phép cache khi build và chia sẻ layer chung giữa nhiều image.

**5. OCI là gì?**
Open Container Initiative — tổ chức dưới Linux Foundation, chuẩn hóa 3 spec: Runtime Spec (cách chạy container), Image Spec (cấu trúc image), Distribution Spec (API push/pull registry). Mục tiêu: image/runtime từ vendor khác nhau vẫn tương thích nhau, tránh khóa chặt vào một sản phẩm.

## Mid-level

**6. Giải thích cụ thể sự khác biệt giữa cgroups v1 và v2.**
cgroups v1: mỗi controller (cpu, memory, io...) có cây phân cấp riêng, mount riêng, một tiến trình có thể ở vị trí khác nhau trong mỗi cây — linh hoạt nhưng khó quản lý nhất quán. cgroups v2: một cây phân cấp thống nhất (unified hierarchy) cho mọi controller, đơn giản hóa quản lý, hỗ trợ Pressure Stall Information (PSI). cgroups v2 là mặc định trên distro hiện đại có systemd ≥ 247, và là kiến trúc được phát triển tiếp — v1 ở chế độ bảo trì.

**7. Cgroup driver `systemd` và `cgroupfs` khác nhau thế nào, tại sao khuyến nghị `systemd`?**
`cgroupfs` là Docker/runc tự thao tác trực tiếp file trong `/sys/fs/cgroup`. `systemd` là giao việc quản lý cho `systemd` (vốn đã là bộ quản lý cgroup chính vì nó là init PID 1). Nếu dùng `cgroupfs` trong khi hệ thống dùng `systemd` làm init, có hai "người quản lý" cùng thao tác một cây cgroup, dễ gây xung đột trạng thái khi `systemd` reload. Production khuyến nghị đồng bộ dùng driver `systemd`.

**8. Giải thích cơ chế Copy-on-Write của OverlayFS.**
`lowerdir` (layer image, read-only) xếp chồng dưới `upperdir` (layer ghi của container, read-write), hợp nhất thành `merged` (view container thực sự dùng). Đọc file chỉ có ở lowerdir: phục vụ trực tiếp, không copy. Ghi/sửa file có ở lowerdir: OverlayFS copy toàn bộ file đó lên upperdir trước (copy-up) rồi mới sửa ở bản copy — file gốc ở lowerdir không bao giờ bị động tới. Xóa file chỉ có ở lowerdir: tạo whiteout file ở upperdir để ẩn nó khỏi view merged.

**9. User namespace giải quyết vấn đề bảo mật gì? Vì sao Docker không bật mặc định?**
User namespace ánh xạ UID 0 (root) bên trong container thành một UID không đặc quyền trên host. Nếu container bị escape do lỗi khác, tiến trình thoát ra chỉ mang UID thường trên host, không phải root thật — giảm mức độ nghiêm trọng của container escape. Docker không bật mặc định (`userns-remap`) vì phá vỡ tương thích với nhiều workload cần chia sẻ UID giữa host và container (ví dụ bind mount volume cần khớp quyền sở hữu file).

**10. Vì sao PID 1 trong container "đặc biệt" hơn PID 1 bình thường trên host?**
PID 1 trong PID namespace đóng vai trò init: phải reap zombie process của tiến trình con, và nếu PID 1 chết thì kernel dọn (kill) toàn bộ các tiến trình còn lại trong cùng namespace đó — tương đương cả container dừng. Ứng dụng không xử lý signal/reap con đúng cách khi làm PID 1 dễ gây ra zombie process tích tụ hoặc không nhận SIGTERM đúng khi `docker stop`.

## Thực chiến

**11. Bạn `docker exec` vào một container để debug nhưng lệnh bash báo "OCI runtime exec failed: exec: bash: no such file or directory". Chẩn đoán và xử lý thế nào?**
Đây không phải lỗi namespace/cgroup — image base (thường Alpine hoặc distroless) không có binary `bash`. Kiểm tra bằng `docker exec <container> which sh` hoặc thử `docker exec -it <container> sh` (Alpine dùng BusyBox `sh`). Với image distroless (không có shell nào), phải debug bằng cách gắn thêm debug container chia sẻ namespace (`docker run --pid=container:<target> --net=container:<target> ... debug-image`) hoặc dùng `docker debug`/ephemeral container tùy container runtime.

**12. Một node Kubernetes báo hàng loạt pod bị CPU throttled dù tổng %CPU node vẫn còn dư nhiều. Container đang chạy gì bên dưới và bạn kiểm tra bằng cách nào?**
Mỗi pod/container có cgroup riêng với `cpu.max` (quota/period) giới hạn CPU dù node còn dư tài nguyên tổng. Kiểm tra `cat /sys/fs/cgroup/.../cpu.stat`, xem `nr_throttled` và `throttled_usec` tăng liên tục — xác nhận container đang chạm trần CPU request/limit đã set quá thấp so với nhu cầu burst thực tế của ứng dụng. Xử lý: tăng CPU limit, hoặc nếu ứng dụng đa luồng, kiểm tra xem nó có tự động set số thread theo `nproc` của **host** (thấy toàn bộ core vật lý qua `/proc/cpuinfo` dù bị cgroup giới hạn) gây tạo quá nhiều thread rồi bị throttle nặng hơn — cần cấu hình ứng dụng đọc đúng giới hạn cgroup thay vì tổng CPU host.

**13. Sau khi nâng cấp distro, `docker info` báo cgroup version chuyển từ 1 sang 2, một số container legacy chạy `--privileged` bắt đầu lỗi liên quan `/sys/fs/cgroup`. Vì sao, và hướng khắc phục?**
Ứng dụng bên trong container (thường monitoring agent cũ hoặc setup tự quản lý cgroup thủ công) code cứng đường dẫn/API theo cấu trúc cgroups v1 (nhiều cây con riêng theo controller), không tương thích với unified hierarchy của v2. Khắc phục: cập nhật ứng dụng/agent lên phiên bản hỗ trợ cgroups v2 (hầu hết công cụ giám sát hiện đại đã hỗ trợ), hoặc tạm thời boot kernel với tham số ép `systemd.unified_cgroup_hierarchy=0` để quay lại cgroups v1/hybrid trong lúc chờ nâng cấp ứng dụng — đây chỉ là giải pháp tạm vì v1 đang ở chế độ bảo trì, không phải hướng lâu dài.

**14. Điều tra sự cố: một container tự nhiên "biến mất" (không còn trong `docker ps -a`) sau khi host bị OOM ở tầng hệ thống (không phải OOM riêng container). Namespace/cgroup nào liên quan, và bạn lấy bằng chứng ở đâu?**
Đây là OOM ở cấp hệ thống (system-wide), không phải cgroup riêng container — kernel OOM-killer chọn tiến trình có điểm `oom_score` cao nhất để giết, có thể là chính tiến trình chính trong container (PID 1 namespace đó), kéo theo containerd-shim dọn dẹp toàn bộ container. Bằng chứng: `dmesg`/`journalctl -k` tìm dòng `Out of memory: Killed process <pid>` (không có chữ "cgroup" như OOM cgroup-scoped), đối chiếu thời điểm với `docker events` hoặc log `containerd`/`dockerd` (`journalctl -u docker`) để xác nhận container nào tương ứng PID đó. Hướng xử lý dài hạn: set `--memory` hợp lý cho từng container để OOM xảy ra ở cấp cgroup (chỉ giết đúng container đó) thay vì để hệ thống phải OOM toàn cục và chọn ngẫu nhiên nạn nhân.
