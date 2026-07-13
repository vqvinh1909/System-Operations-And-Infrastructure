---
title: "07 - Interview"
module: 9
tags: [docker, sysops-infra, module-09, interview]
---

# 07 - Interview: Docker Engine

## Junior

**1. Docker Engine gồm những thành phần nào?**
Docker CLI (client gõ lệnh), `dockerd` (daemon quản lý image/network/volume, nhận request qua REST API), `containerd` (quản lý vòng đời container, giao tiếp với `dockerd` qua gRPC), `runc` (OCI runtime, thực sự gọi syscall tạo namespace/cgroup rồi thoát).

**2. `docker stop` và `docker kill` khác nhau thế nào?**
`stop` gửi SIGTERM, chờ grace period (mặc định 10 giây) cho tiến trình tự dọn dẹp và thoát, hết giờ mới gửi SIGKILL. `kill` gửi SIGKILL ngay lập tức, không có thời gian dọn dẹp.

**3. Docker daemon lắng nghe ở đâu theo mặc định?**
Unix domain socket `/var/run/docker.sock` (`/run/docker.sock`). Có thể cấu hình thêm TCP socket nhưng mặc định không bật TLS trên đó — rủi ro bảo mật nếu bind ra `0.0.0.0` không kiểm soát.

**4. Vì sao thêm user vào nhóm `docker` được coi là cấp quyền root?**
Vì `dockerd` chạy với quyền root và mọi request qua socket (mà thành viên nhóm `docker` truy cập được không cần sudo) đều được daemon thực thi với quyền root — ví dụ mount toàn bộ filesystem host vào container rồi chroot vào là lấy được root shell trên host.

**5. `docker create` và `docker run` khác nhau thế nào?**
`docker create` chỉ tạo container (cấp ID, chuẩn bị filesystem/network) ở trạng thái `created`, chưa chạy tiến trình entrypoint. `docker run` = `create` + `start` gộp lại, thực sự chạy tiến trình.

## Mid-level

**6. Giải thích vai trò của `containerd-shim`, vì sao nó cần tồn tại.**
Shim là tiến trình cha thực sự của container process (không phải `containerd` hay `runc`). `runc` chỉ tạo container rồi thoát ngay, không ở lại làm cha. Shim đăng ký làm subreaper (`prctl(PR_SET_CHILD_SUBREAPER)`) để nhận nuôi container process, giữ nó sống độc lập với `containerd`/`dockerd` — nhờ vậy container không bị ảnh hưởng khi restart/upgrade daemon, và containerd không phải tự làm cha trực tiếp cho hàng trăm container cùng lúc.

**7. `dockerd` và `containerd` giao tiếp qua giao thức gì? Vì sao khác với CLI-daemon?**
`dockerd` → `containerd` qua gRPC (qua socket `/run/containerd/containerd.sock`), hiệu năng cao, hướng nội bộ hệ thống. CLI → `dockerd` qua REST API trên Unix socket, hướng người dùng cuối/công cụ ngoài. Hai giao thức phục vụ hai mục đích khác nhau: REST dễ tích hợp rộng rãi, gRPC tối ưu cho giao tiếp nội bộ tần suất cao.

**8. Vì sao Kubernetes hiện đại dùng thẳng `containerd` mà không cần `dockerd`?**
`containerd` độc lập với `dockerd`, có thể phục vụ trực tiếp qua CRI (Container Runtime Interface) mà Kubernetes gọi tới. Từ khi Dockershim bị loại khỏi Kubernetes, kubelet giao tiếp thẳng với `containerd` (hoặc CRI-O), bỏ qua tầng `dockerd` vốn không cần thiết cho việc chỉ chạy container theo lệnh của kubelet.

**9. `docker pause` hoạt động theo cơ chế nào, khác gì so với `stop`?**
`pause` dùng cgroup freezer (`cgroup.freeze` trong cgroups v2) để đông cứng toàn bộ tiến trình trong container ở tầng lập lịch CPU của kernel — tiến trình vẫn còn nguyên trong bộ nhớ, không nhận được tín hiệu nào, hoàn toàn không biết mình bị dừng. `stop` gửi SIGTERM thực sự, tiến trình biết và có cơ hội dọn dẹp trước khi thoát hẳn.

**10. Registry Distribution Spec giải quyết vấn đề gì trong luồng `docker pull`?**
Chuẩn hóa API push/pull (manifest, blob layer) để bất kỳ registry nào tuân thủ (Docker Hub, Harbor, ECR...) đều tương tác được với bất kỳ client nào tuân thủ. Luồng pull: lấy manifest (danh sách layer digest) → kiểm tra layer nào đã có cục bộ theo digest (tránh tải trùng) → chỉ tải layer thiếu → ghép layer bằng storage driver thành image.

## Thực chiến

**11. Một node production sau khi `apt upgrade docker-ce` bị mất kết nối daemon trong 45 giây, nhưng container không bị restart. Giải thích hiện tượng và đánh giá đây có phải sự cố nghiêm trọng không.**
Đây là hành vi bình thường, không phải sự cố: `dockerd` restart trong quá trình upgrade khiến API tạm thời không phản hồi (không `docker ps` được), nhưng nhờ kiến trúc `containerd` + `containerd-shim` độc lập, các container process vẫn tiếp tục chạy suốt thời gian đó — chỉ là "mất khả năng quản lý qua CLI" tạm thời, không phải downtime của ứng dụng. Xác nhận bằng `docker inspect --format '{{.State.StartedAt}}'` — thời điểm start container không đổi qua sự kiện upgrade, chứng minh tiến trình không bị khởi động lại.

**12. Sau một đợt audit bảo mật, team yêu cầu loại bỏ hoàn toàn việc cấp quyền nhóm `docker` cho CI/CD runner nhưng pipeline vẫn cần build và chạy container. Đề xuất hướng giải quyết.**
Vấn đề gốc là mount `docker.sock` vào container CI (Docker-in-Docker qua socket) tương đương cấp quyền root host cho pipeline. Hướng thay thế: (1) dùng rootless Docker cho runner nếu vẫn cần Docker CLI đầy đủ nhưng không cần daemon chạy full quyền root; (2) dùng công cụ build không cần daemon như Kaniko/BuildKit standalone chạy trong container không đặc quyền, đặc biệt phù hợp nếu runner chạy trên Kubernetes; (3) tách hẳn máy build ra khỏi runner CI dùng chung, build trên máy/VM riêng có kiểm soát quyền chặt hơn.

**13. `docker info` báo `Cgroup Driver: cgroupfs`, node này sắp join vào cụm Kubernetes dùng kubelet cấu hình cgroup driver `systemd`. Rủi ro là gì và cách khắc phục trước khi join?**
Không khớp cgroup driver giữa container runtime và kubelet gây xung đột quản lý cgroup (hai bộ quản lý cùng thao tác một cây, dễ dẫn tới giới hạn tài nguyên không nhất quán hoặc pod bị lỗi khởi động khó chẩn đoán). Khắc phục: sửa `/etc/docker/daemon.json` thêm `"exec-opts": ["native.cgroupdriver=systemd"]`, `systemctl restart docker`, xác nhận lại `docker info | grep -i "cgroup driver"` trước khi cho node join cụm — không nên đổi driver sau khi node đã có container/pod đang chạy vì có thể cần restart lại toàn bộ workload trên node đó.

**14. Ứng dụng team dev báo container tự nhiên restart liên tục dù không ai `docker restart`, `docker ps -a` cho thấy `RestartCount` tăng dần. Quy trình điều tra bằng kiến trúc đã học ở module này thế nào?**
Kiểm tra `docker inspect --format '{{.HostConfig.RestartPolicy}}'` xác nhận policy đang set (`always`/`unless-stopped`/`on-failure`) — đây là nguyên nhân container tự khởi động lại sau khi exit, không phải hành động thủ công. Tiếp theo kiểm tra `docker inspect --format '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}}'` của lần chạy trước đó để tìm lý do container exit ban đầu (OOM, lỗi ứng dụng, healthcheck fail), và `docker logs --since <thời điểm trước đó>` để xem log ngay trước lần exit gần nhất. Gốc rễ vấn đề luôn nằm ở lý do container exit lần đầu — restart policy chỉ là cơ chế che giấu triệu chứng, không phải nguyên nhân.
