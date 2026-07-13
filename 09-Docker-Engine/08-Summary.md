---
title: "08 - Summary"
module: 9
tags: [docker, sysops-infra, module-09, summary]
---

# 08 - Summary: Docker Engine

## Những gì đã học

- **Kiến trúc client-server**: Docker CLI là client, `dockerd` là server thực sự thao tác image/network/volume, giao tiếp qua REST API trên Unix socket `/var/run/docker.sock`.
- **Chuỗi 4 tầng**: `dockerd` (quản lý cấp cao) → `containerd` (vòng đời container, gRPC) → `containerd-shim` (giữ container sống độc lập, subreaper) → `runc` (OCI runtime, gọi syscall tạo namespace/cgroup rồi thoát ngay).
- **Registry**: Docker Hub/private registry lưu trữ image theo chuẩn OCI Distribution Spec; luôn pin tag cụ thể (hoặc digest) trong production, tránh `latest`.
- **daemon.json**: cấu hình log driver/log-opts (giới hạn dung lượng log bắt buộc), cgroup driver `systemd`, registry-mirrors, live-restore.
- **Rủi ro bảo mật nhóm `docker`**: tương đương cấp quyền root trên host — cần kiểm soát chặt, cân nhắc rootless Docker cho môi trường nhạy cảm.
- **Vòng đời container**: created → running → (paused) → exited → removed. `stop` (SIGTERM + grace period) khác `kill` (SIGKILL ngay) khác `pause` (cgroup freezer, không tín hiệu).

## Checklist tự đánh giá

- [ ] Vẽ được sơ đồ 4 tầng `dockerd → containerd → shim → runc` và giải thích vì sao tách thành nhiều tiến trình.
- [ ] Giải thích được vì sao container sống sót qua việc restart `dockerd`.
- [ ] Cấu hình được `daemon.json` với log giới hạn dung lượng và cgroup driver đúng.
- [ ] Giải thích chính xác rủi ro bảo mật của nhóm `docker`, không hạ thấp mức độ nghiêm trọng.
- [ ] Phân biệt đúng ngữ cảnh dùng `stop`/`kill`/`pause` trong vận hành thực tế.
- [ ] Đọc `docker inspect` để chẩn đoán exit code, OOMKilled, RestartPolicy khi troubleshoot.

## Liên kết tới Module tiếp theo

Module này tập trung vào **cách Docker Engine chạy container từ một image có sẵn**. [[../10-Docker-Images/README|Module 10 - Docker Images]] sẽ đi ngược lại một bước: image đó **được tạo ra như thế nào** — cú pháp Dockerfile, cơ chế build và cache layer (liên hệ trực tiếp Image Layer đã học ở Module 08), multi-stage build, BuildKit, và các thực hành tối ưu kích thước/bảo mật image. Hiểu rõ vòng đời container và kiến trúc engine ở module này là nền tảng bắt buộc để hiểu tại sao một số quyết định khi viết Dockerfile (ví dụ thứ tự lệnh, `USER` non-root) lại ảnh hưởng trực tiếp tới cách container chạy sau này.

> [!tip] Ghi nhớ cốt lõi
> `docker run` không phải một hộp đen — nó là một chuỗi bàn giao rõ ràng qua REST API, gRPC, fork/exec, và cuối cùng là syscall kernel. Mỗi tầng tồn tại vì một lý do kiến trúc cụ thể (tách trách nhiệm, giảm blast radius, chuẩn hóa OCI) — nhớ đúng lý do, không chỉ nhớ tên các tầng.
