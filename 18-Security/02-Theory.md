---
title: "02 - Theory"
module: 18
tags: [docker, sysops-infra, module-18, security, rootless, seccomp, apparmor]
---

# 02 — Lý thuyết: Rootless, Capabilities, Seccomp, AppArmor, Image Scanning

## 1. Vì sao Docker mặc định (rootful) là một rủi ro kiến trúc

Ở chế độ mặc định, `dockerd` (Docker daemon) chạy bằng quyền **root (UID 0)** trên host. Mọi container được daemon đó tạo ra, về bản chất, là tiến trình con của một tiến trình root. Docker dùng namespace (đã học ở Part II) để **cô lập** những gì container nhìn thấy (PID, network, mount, hostname riêng...), nhưng namespace **không phải là một bức tường bảo mật tuyệt đối** — nó là cơ chế cô lập, không phải cơ chế phân quyền triệt để. Nếu có lỗ hổng ở kernel hoặc ở chính runtime container, một tiến trình thoát được khỏi namespace vẫn có thể chạm tới quyền root thật trên host — vì về bản chất UID 0 bên trong container, nếu không dùng thêm user namespace remapping, **chính là** UID 0 trên host.

## 2. Rootless Docker — chạy cả daemon lẫn container bằng user thường

**Rootless Docker** giải quyết đúng vấn đề trên bằng cách chạy toàn bộ `dockerd` bằng một **user thường** (không phải root), không chỉ container bên trong nó. Cơ chế internal working:

- Docker rootless dùng **user namespace** ngay từ tầng daemon: UID/GID bên trong "thế giới" của daemon được ánh xạ (remap) tới một dải UID/GID không đặc quyền trên host (thường cấu hình qua `/etc/subuid`, `/etc/subgid`).
- Vì daemon không chạy bằng root, nó **không có quyền tạo network bridge/veth pair theo cách thông thường** (việc này vốn cần quyền root) — Rootless Docker giải quyết bằng cách dùng công cụ **RootlessKit** kết hợp một trong các phương án networking không cần quyền root như `slirp4netns` hoặc `VPNKit`, đánh đổi lấy một phần hiệu năng mạng (đóng gói/giải gói thêm một tầng so với bridge network thật).
- Hệ quả bảo mật: nếu kẻ tấn công thoát được khỏi container chạy trong Rootless Docker, thứ họ chạm tới **không phải root thật của host** — chỉ là UID của user thường đã khởi chạy Rootless daemon đó, giảm đáng kể mức độ thiệt hại có thể gây ra.

> [!info] Đánh đổi cần biết
> Rootless Docker có một số giới hạn: hiệu năng network thấp hơn một chút do lớp `slirp4netns`, một số tính năng cần quyền root thật (ví dụ một số driver storage, một số cấu hình network nâng cao) không khả dụng hoặc cần cấu hình đặc biệt. Đây là lý do cần đánh giá theo từng workload cụ thể, không áp dụng máy móc cho mọi trường hợp — xu hướng ngành năm 2026 nghiêng về mặc định rootless-first cho môi trường dev/CI và các workload single-node, trong khi production Kubernetes quy mô lớn vẫn đang trong giai đoạn hoàn thiện dần.

## 3. Capabilities — chia nhỏ quyền root thành từng mảnh

Trên Linux, "quyền root" không phải một khối nguyên vẹn — kernel chia quyền root thành khoảng 40 **capability** độc lập (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_CHOWN`, `CAP_SETUID`...), mỗi capability cho phép thực hiện một nhóm hành động cụ thể vốn trước đây chỉ root mới làm được.

Docker mặc định **không** cấp toàn bộ capability cho container (khác với chạy tiến trình bằng root trực tiếp trên host) — nó chỉ cấp một tập con khoảng 14 capability được coi là "an toàn tương đối" và cần thiết cho phần lớn ứng dụng thông thường (ví dụ `CAP_CHOWN`, `CAP_SETUID`, `CAP_NET_BIND_SERVICE`...). Tuy nhiên, tập mặc định này **vẫn còn dư thừa** so với nhu cầu thật của rất nhiều ứng dụng — nguyên tắc "principle of least privilege" (đã học ở Security Hardening khóa trước) yêu cầu bạn thu hẹp hơn nữa.

Thực hành chuẩn:
```
--cap-drop=ALL        # bo het moi capability mac dinh
--cap-add=<ten>       # chi cap lai dung capability ma ung dung THAT SU can
```

Một ứng dụng web thông thường (không cần bind port < 1024, không cần thao tác network nâng cao, không cần đổi UID lúc runtime) thường **không cần bất kỳ capability nào thêm** sau khi `--cap-drop=ALL` — đây là trường hợp lý tưởng và phổ biến hơn bạn nghĩ trong thực tế.

## 4. Seccomp — chặn ở tầng syscall

**Seccomp** (secure computing mode) là cơ chế của kernel Linux giới hạn tập **system call (syscall)** mà một tiến trình được phép gọi. Một image ứng dụng thông thường chỉ cần một tập syscall rất nhỏ trong tổng số hơn 300 syscall mà kernel Linux hỗ trợ — phần lớn syscall còn lại (ví dụ liên quan trực tiếp tới quản trị kernel, module, mount hệ thống...) không bao giờ được ứng dụng bình thường gọi tới, nhưng lại là công cụ ưa thích của mã độc/exploit.

Docker áp dụng một **seccomp profile mặc định** cho mọi container, chặn khoảng 44 syscall nguy hiểm trong khi vẫn cho phép mọi thứ một ứng dụng thông thường cần. Bạn có thể viết seccomp profile tùy chỉnh (dạng JSON, liệt kê syscall được phép/bị chặn) để siết chặt hơn nữa cho một ứng dụng cụ thể, nhưng đây là việc tốn công bảo trì — khuyến nghị chung năm 2026 là **giữ nguyên profile mặc định của Docker** (không dùng `--security-opt seccomp=unconfined`), chỉ viết profile tùy chỉnh khi có bằng chứng rõ ràng cần thiết.

## 5. AppArmor — chặn ở tầng truy cập tài nguyên (file, mạng, capability)

Bạn đã học AppArmor ở khóa Linux Sysadmin để giới hạn một tiến trình trên host chỉ được đọc/ghi những đường dẫn file cụ thể, dù tiến trình đó có chạy bằng root. Docker áp dụng đúng nguyên lý đó cho container: mặc định gán một **AppArmor profile tên `docker-default`** cho mọi container mới tạo (nếu host có AppArmor và kernel hỗ trợ) — profile này giới hạn container chỉ truy cập những đường dẫn/tài nguyên cần thiết, chặn nhiều hành vi nguy hiểm dù capability hay seccomp có vô tình cho phép.

**Seccomp và AppArmor bổ sung cho nhau, không thay thế nhau:**

| | Seccomp | AppArmor |
|---|---|---|
| Chặn ở mức | System call (hành động gọi tới kernel) | File path, network, capability sử dụng thực tế |
| Ví dụ chặn | Gọi `mount()`, `ptrace()` | Đọc `/etc/shadow`, ghi ngoài thư mục cho phép |
| Câu hỏi trả lời | "Có được gọi hành động này không?" | "Có được chạm vào tài nguyên này không?" |

Thực hành chuẩn (đã xác nhận qua WebSearch 2026): dùng cả hai cùng lúc ở chế độ mặc định (enforcing), không tắt cái nào, chỉ tùy biến khi có lý do cụ thể — đây gọi là defense in depth, xem sơ đồ minh họa ở [[03-Architecture]].

## 6. Image Scanning — chặn lỗ hổng trước khi nó chạy

Đã giới thiệu khái niệm ở Module 16 (Image Registry) khi nói về Trivy tích hợp trong Harbor. Ở module này, ta đào sâu hơn về **quy trình** đưa scanning vào vòng đời phát triển:

- Scan không chỉ chạy một lần lúc build — cần **scan lại định kỳ** image đang chạy production, vì CVE mới liên tục được công bố cho các package đã tồn tại sẵn trong image từ trước.
- Kết quả scan phân loại theo mức độ nghiêm trọng (thường theo CVSS: `Critical`, `High`, `Medium`, `Low`) — một pipeline CI/CD trưởng thành cần **gate** tự động: build fail nếu phát hiện CVE mức Critical/High chưa có bản vá, thay vì chỉ ghi log cảnh báo không ai đọc.
- Công cụ phổ biến: **Trivy** (mã nguồn mở, tích hợp sẵn trong nhiều registry doanh nghiệp bao gồm Harbor), **Docker Scout** (tích hợp sẵn trong Docker CLI, đã thay thế `docker scan` cũ dựa trên Snyk).

## 7. Docker Bench for Security — tự chấm điểm hệ thống theo CIS Benchmark

**Docker Bench for Security** (`docker/docker-bench-security`) là một script mã nguồn mở do Docker duy trì, tự động kiểm tra một hệ thống Docker đang chạy so với **CIS Docker Benchmark** — một bộ khuyến nghị bảo mật chuẩn hóa do Center for Internet Security xuất bản, bao phủ nhiều nhóm: cấu hình host, cấu hình daemon, cấu hình image/container, thực hành vận hành runtime.

Script chạy và in ra danh sách các mục `[PASS]`/`[WARN]`/`[INFO]` tương ứng từng khuyến nghị trong benchmark — ví dụ kiểm tra daemon có bật audit logging chưa, container có đang chạy với `--privileged` không cần thiết không, image có build từ user root không thay vì user riêng.

> [!warning] Lưu ý xác nhận tính đến 07/2026
> Repository `docker/docker-bench-security` vẫn đang được duy trì trên GitHub và hỗ trợ CIS Docker Benchmark v1.6.0, nhưng image dựng sẵn (`docker/docker-bench-security` trên Docker Hub) được cộng đồng ghi nhận là **lỗi thời**. Thực hành đúng năm 2026 là **clone repository và tự build/chạy trực tiếp bằng script `docker-bench-security.sh`**, không phụ thuộc image build sẵn không rõ đã cập nhật gần đây hay chưa — xem lệnh cụ thể ở [[04-Commands]].
