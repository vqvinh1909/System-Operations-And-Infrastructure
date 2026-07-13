---
title: "03 - Architecture"
module: 18
tags: [docker, sysops-infra, module-18, security, architecture]
---

# 03 — Kiến trúc: Rootful vs Rootless, Phòng thủ theo lớp

> [!note] Cách đo sơ đồ trong file này
> Mọi khung (box) dưới đây được dựng bằng script Python nội bộ (hàm `box()` tính `inner_width = max(len(dòng)) + padding*2`, dùng đúng `inner_width` đó cho border trên, border dưới, và mọi dòng nội dung), sau đó chạy `verify_boxes.py` xác nhận `len()` khớp nhau cho từng khung — không đếm tay.

## Sơ đồ 1: Docker Rootful — đường đi của quyền root

```
+---------------+
| USER (thuong) |
+---------------+
  | docker run
  v
+---------------+
| DOCKER CLIENT |
+---------------+
  | goi API qua socket
  v
+--------------------------------+
| DOCKERD chay bang ROOT (UID 0) |
+--------------------------------+
  | fork container
  v
+------------------------------------------+
| CONTAINER PROCESS                        |
| neu thoat container -> UID 0 tren HOST   |
| (rui ro: container breakout = root host) |
+------------------------------------------+
```

Kể cả khi user gõ lệnh `docker run` chỉ là một user thường, socket Docker (`/var/run/docker.sock`) giao tiếp thẳng với một daemon chạy bằng root — đây là lý do "quyền chạy được lệnh `docker`" tương đương "quyền root trên host" trong mô hình rootful, một sự thật thường bị đánh giá thấp khi cấp quyền thành viên nhóm `docker` cho nhân sự.

## Sơ đồ 2: Docker Rootless — đường đi bị chặn ở đúng tầng cần chặn

```
+----------------------------------+
| USER thuong (vd: vinh, UID 1000) |
+----------------------------------+
  | docker run
  v
+---------------+
| DOCKER CLIENT |
+---------------+
  | goi API qua socket rieng cua user
  v
+----------------------------------------------+
| DOCKERD chay bang UID 1000 (khong phai root) |
+----------------------------------------------+
  | fork container qua user namespace
  v
+--------------------------------------------------+
| CONTAINER PROCESS                                |
| neu thoat container -> chi la UID 1000 tren HOST |
| (khong co quyen root that -> giam thiet hai)     |
+--------------------------------------------------+
```

So sánh trực tiếp với Sơ đồ 1: cấu trúc luồng đi giống hệt nhau (User → Client → Daemon → Container), khác biệt duy nhất nhưng mang tính quyết định nằm ở **UID mà daemon chạy bằng**. Đây chính là lý do Rootless Docker được coi là một thay đổi kiến trúc, không phải một cờ cấu hình phụ — nó thay đổi toàn bộ "trần" quyền hạn mà mọi thứ phía dưới có thể chạm tới.

## Sơ đồ 3: Mô hình phòng thủ theo lớp (Defense in Depth) áp dụng cho một container

```
+-------------------------------------------+
| LOP 1 - IMAGE SCAN (Trivy / Docker Scout) |
| chan CVE truoc khi build len registry     |
+-------------------------------------------+
  |
  v
+-----------------------------------------+
| LOP 2 - ROOTLESS / USER namespace       |
| container khong chay bang UID 0 that su |
+-----------------------------------------+
  |
  v
+--------------------------------------------+
| LOP 3 - CAPABILITIES                       |
| --cap-drop=ALL, chi --cap-add dung thu can |
+--------------------------------------------+
  |
  v
+--------------------------------------------+
| LOP 4 - SECCOMP                            |
| chan syscall nguy hiem (vd: ptrace, mount) |
+--------------------------------------------+
  |
  v
+-------------------------------------------------+
| LOP 5 - APPARMOR                                |
| gioi han file/duong dan container duoc truy cap |
+-------------------------------------------------+
  |
  v
+-------------------------------+
| CONTAINER PROCESS DUOC BAO VE |
+-------------------------------+
```

Nguyên tắc cốt lõi của defense in depth: **không lớp nào được thiết kế để hoàn hảo 100%**. Lớp 1 (image scan) có thể bỏ sót một CVE zero-day chưa được công bố. Nếu kẻ tấn công vượt qua được lớp 1, họ vẫn chạm phải lớp 2 (không có quyền root thật dù chiếm được container). Nếu bằng cách nào đó vẫn cố thực hiện hành vi nguy hiểm, họ chạm lớp 3 (capability bị thu hẹp khiến nhiều hành động không thể gọi). Nếu vẫn cố gọi syscall nguy hiểm, lớp 4 (seccomp) chặn ở tầng kernel. Cuối cùng, nếu vẫn cố truy cập file/tài nguyên ngoài phạm vi cho phép, lớp 5 (AppArmor) chặn lại. **Chỉ khi vượt qua đồng thời cả 5 lớp**, một cuộc tấn công mới thực sự gây thiệt hại nghiêm trọng ra ngoài phạm vi container — xác suất đó thấp hơn rất nhiều so với việc chỉ dựa vào một lớp duy nhất.

## Sơ đồ 4: Image Scanning như một Gate trong pipeline

```
+--------------------------+
| Dockerfile (source code) |
+--------------------------+
  | docker build
  v
+------------------------------+
| IMAGE moi build xong (local) |
+------------------------------+
  | trivy image / docker scout cves
  v
+-------------------------------+
| SCAN CVE + SECRET + MISCONFIG |
+-------------------------------+
  |
  v
+-----------------------------------+
| GATE: co CVE Critical/High khong? |
+-----------------------------------+
```

Sau bước `GATE` ở trên, pipeline rẽ nhánh theo kết quả:
- **Không có CVE Critical/High** (hoặc đã whitelist rõ ràng có lý do) → tiếp tục `docker push` lên registry, cho phép deploy.
- **Có CVE Critical/High chưa xử lý** → build **fail ngay tại đây**, không cho image đi tiếp tới bước push/deploy, đồng thời thông báo cho team qua kênh CI/CD (Slack, email...). Đây là điểm khác biệt giữa "có scan" và "scan có tác dụng thật" — scan chỉ ghi log mà không chặn được coi là hình thức, không phải kiểm soát bảo mật thật sự.

> [!info] Liên hệ Security Hardening đã học ở Linux Sysadmin
> Cả 5 lớp phòng thủ ở Sơ đồ 3 đều là biến thể container hóa của những nguyên tắc bạn đã học: least privilege (Lớp 2, 3), giảm bề mặt tấn công bằng cách chặn hành vi không cần thiết (Lớp 4 tương đương SELinux/AppArmor mức host bạn đã cấu hình cho tiến trình hệ thống), và "shift left" bảo mật — xử lý lỗ hổng càng sớm càng rẻ (Lớp 1, tương đương patch management định kỳ ở mức host).
