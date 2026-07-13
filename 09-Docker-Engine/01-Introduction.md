---
title: "01 - Introduction"
module: 9
tags: [docker, sysops-infra, module-09, introduction]
---

# 01 - Giới thiệu Docker Engine

## Docker Engine là gì

"Docker" mà bạn gõ trên terminal thực chất là tên gọi chung cho một **hệ thống nhiều thành phần phối hợp**, không phải một chương trình duy nhất. Cụm từ chính xác là **Docker Engine**, bao gồm:

- **Docker CLI** (`docker`) — công cụ dòng lệnh bạn gõ.
- **Docker Daemon** (`dockerd`) — tiến trình nền chạy trên server, thực sự quản lý image, container, network, volume.
- **containerd** — một tiến trình nền khác, đứng giữa `dockerd` và runtime cấp thấp, quản lý vòng đời container ở mức OS.
- **runc** — công cụ dòng lệnh tuân theo chuẩn OCI (Open Container Initiative), là bên thực sự gọi các system call của kernel Linux (namespace, cgroup) để tạo ra container process.

Đây không phải chi tiết vụn vặt. Ở Module 08 bạn học rằng "container" về bản chất là một tiến trình Linux bình thường được cô lập bằng namespace/cgroup. Module 09 trả lời câu hỏi tiếp theo: **ai** ra lệnh cho kernel tạo namespace/cgroup đó, và **bằng cách nào** một lệnh gõ trên CLI đi xuyên qua nhiều tầng phần mềm để tới được kernel.

## Tại sao phải tách thành nhiều tiến trình thay vì một chương trình duy nhất

Đây là câu hỏi quan trọng cho tư duy kiến trúc, không chỉ để "biết cho vui":

1. **Tách trách nhiệm (separation of concerns)**: `dockerd` lo image, network, volume, API cho người dùng cuối. `containerd` chỉ lo vòng đời container (start/stop/thu thập trạng thái) và có thể phục vụ nhiều client khác ngoài Docker (Kubernetes qua CRI dùng thẳng containerd, không qua `dockerd`). `runc` chỉ lo đúng một việc: tạo 1 container theo đúng chuẩn OCI rồi thoát.
2. **Giảm blast radius khi crash/restart**: Nếu toàn bộ logic nằm trong một tiến trình duy nhất, restart hoặc crash tiến trình đó đồng nghĩa mọi container đang chạy cũng chết theo (vì container process là con của tiến trình đó). Tách `containerd` ra khỏi `dockerd`, và đặc biệt tách `containerd-shim` ra khỏi cả hai, cho phép **container tiếp tục chạy dù `dockerd` bị restart hoặc upgrade**. Đây là lý do thực tế doanh nghiệp rất quan tâm: bạn có thể `apt upgrade docker-ce` mà không làm rớt các container production đang chạy.
3. **Chuẩn hóa để tương tác được với hệ sinh thái**: `runc` tuân theo OCI Runtime Spec — bất kỳ công cụ nào tuân theo chuẩn này (kể cả không phải Docker, ví dụ Podman, CRI-O) đều có thể chạy container tương thích. Đây là lý do Docker "hiến tặng" containerd và runc cho Cloud Native Computing Foundation (CNCF) — biến chúng thành hạ tầng dùng chung của cả ngành, không riêng gì Docker.

## Bối cảnh lịch sử ngắn gọn (để hiểu vì sao kiến trúc như hiện tại)

Ban đầu Docker là một khối nguyên (monolith): `docker` daemon tự làm hết mọi việc từ quản lý image tới gọi trực tiếp LXC (Linux Containers) để tạo container. Theo thời gian, để chuẩn hóa và mở hệ sinh thái, phần lõi chạy container được tách ra thành `runc` (dựa trên OCI Runtime Spec do chính Docker khởi xướng cùng cộng đồng), và phần quản lý vòng đời container được tách ra thành `containerd`. Kubernetes cũng hưởng lợi trực tiếp: kể từ khi Dockershim bị loại bỏ khỏi Kubernetes, nhiều cụm Kubernetes hiện đại dùng thẳng `containerd` làm container runtime thông qua CRI (Container Runtime Interface) mà không cần `dockerd` ở giữa — minh chứng containerd đã trở thành thành phần hạ tầng độc lập, không còn "thuộc về" riêng Docker.

## Mục tiêu học tập của module

Sau khi hoàn thành module này, bạn có thể:

1. Vẽ và giải thích luồng gọi đầy đủ từ `docker run` tới tiến trình container thực sự chạy trong kernel.
2. Giải thích vai trò riêng biệt của `dockerd`, `containerd`, `containerd-shim`, `runc`.
3. Cấu hình `daemon.json` đúng cách và biết hậu quả của từng tùy chọn phổ biến.
4. Đánh giá đúng mức độ rủi ro bảo mật của việc thêm user vào nhóm `docker`.
5. Thao tác thành thạo vòng đời container: create, start, stop, pause/unpause, kill, remove — và biết chọn đúng lệnh cho đúng tình huống.
6. Chẩn đoán các lỗi vận hành phổ biến liên quan tới daemon, socket, và trạng thái container.

## Liên hệ với môn học đã học

Bạn đã học Ansible song song — hãy để ý: các module tự động hóa triển khai Docker bằng Ansible (ví dụ dùng module `community.docker.docker_container`) thực chất cũng chỉ là gọi REST API của `dockerd` giống hệt Docker CLI, chỉ khác ở chỗ Ansible gọi qua HTTP client thay vì qua CLI. Hiểu kiến trúc ở module này giúp bạn đọc hiểu behaviour của các module Ansible đó dễ dàng hơn nhiều.

Tiếp theo: đọc [[02-Theory]] để đi sâu vào cơ chế hoạt động bên trong từng thành phần.
