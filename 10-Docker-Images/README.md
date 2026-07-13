---
title: "Module 10 - Docker Images"
tags: [docker, sysops-infra, module-10, dockerfile, image-build]
module: 10
part: "II - Container Fundamentals"
difficulty: Intermediate
status: draft
created: 2026-07-12
prerequisites: ["[[../09-Docker-Engine/README|Module 09]]"]
next: "[[../11-Container-Runtime/README|Module 11]]"
---

# Module 10 - Docker Images

## Module này nói về gì

Module 09 đã giải thích Docker Engine hoạt động ra sao — daemon, client, API, container runtime bên dưới. Nhưng daemon chỉ chạy được container khi có một thứ làm "khuôn": **image**. Module 10 đi sâu vào cách image được **xây dựng** — từ Dockerfile, qua build engine (BuildKit), ra một artifact bất biến (immutable) mà bất kỳ Docker host nào cũng chạy được giống hệt nhau.

Đây là kỹ năng bạn dùng **hàng ngày** khi làm DevOps/SysAdmin: viết Dockerfile cho app nội bộ, tối ưu kích thước image để pipeline CI/CD chạy nhanh hơn, và debug khi image "tự nhiên" build ra khác kết quả trên máy đồng nghiệp.

## Vì sao module này quan trọng với công việc thực tế

Trong doanh nghiệp, image là đơn vị bàn giao (deliverable) giữa dev và ops. Một image viết ẩu — base image nặng 1.2GB, chạy bằng root, cache miss liên tục — sẽ kéo chậm toàn bộ CI/CD pipeline, tốn băng thông kéo image trên mọi node Kubernetes/Swarm, và mở rộng bề mặt tấn công (attack surface) không cần thiết. Ngược lại, một Dockerfile viết đúng kỹ thuật (multi-stage, cache layer hợp lý, non-root user) là dấu hiệu rõ nhất phân biệt một SysAdmin/DevOps junior với người có kinh nghiệm thực chiến.

## Mục tiêu học tập

Sau module này, bạn có thể:

- Viết Dockerfile đúng chuẩn với đầy đủ instruction quan trọng, hiểu rõ khi nào dùng cái nào.
- Giải thích cơ chế layer cache của Docker và chủ động sắp xếp Dockerfile để tối đa cache hit.
- Áp dụng multi-stage build để giảm kích thước image production xuống mức tối thiểu.
- Sử dụng BuildKit (cache mount, secret mount, buildx) để build nhanh hơn và an toàn hơn (không leak secret vào layer).
- Áp dụng best practice bảo mật và tối ưu image dùng trong môi trường doanh nghiệp thực tế.
- Chẩn đoán và xử lý các lỗi build thường gặp: cache không hit, image phình to, COPY sai context, "no space left on device".

## Cấu trúc nội dung

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Giới thiệu tổng quan, image là gì, vì sao cần hiểu sâu |
| [[02-Theory]] | Lý thuyết chi tiết: Dockerfile instruction, layer, cache, Internal Working |
| [[03-Architecture]] | Sơ đồ ASCII: layer stacking, multi-stage build, luồng BuildKit |
| [[04-Commands]] | Lệnh `docker build`, cú pháp, Dockerfile mẫu đầy đủ, multi-stage, cache mount |
| [[05-Labs]] | Bài lab Basic -> Intermediate -> Advanced/Mini Project |
| [[06-Troubleshooting]] | Lỗi thường gặp và cách xử lý thực chiến |
| [[07-Interview]] | Câu hỏi phỏng vấn: Junior / Mid-level / Thực chiến |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 11 |
| [[labs/README|labs/]] | Nơi lưu code/config thực hành |
| [[scripts/README|scripts/]] | Nơi lưu script tự viết |

## Điều kiện tiên quyết

- Đã hoàn thành [[../09-Docker-Engine/README|Module 09 - Docker Engine]] (hiểu daemon, client, container lifecycle cơ bản).
- Biết Linux filesystem, permission, process cơ bản (từ chuỗi 29 module Linux Sysadmin đã học).
- Có Docker Engine (bản 29.x) cài sẵn để thực hành lab.

## Bước tiếp theo

Sau khi build được image, bước tiếp theo là hiểu **runtime** thực sự chạy container từ image đó như thế nào ở tầng thấp — namespace, cgroups, container runtime (runc, containerd). Đó là nội dung [[../11-Container-Runtime/README|Module 11 - Container Runtime]].

---

> [!note] Self-Review
> - Đã WebSearch kiểm tra: phiên bản Docker Engine hiện hành (29.6.1, phát hành 26/6/2026) và cú pháp `RUN --mount=type=cache` của BuildKit (yêu cầu dòng `# syntax=docker/dockerfile:1`, ví dụ cache apt/pip/npm/go/maven) — khớp với facts sheet đã cung cấp, không phát sinh số liệu bịa.
> - Cả 3 sơ đồ ASCII trong [[03-Architecture]] (layer cache stack, multi-stage build, luồng BuildKit) đã được đo bằng script Python đo độ dài từng dòng khung — kết quả toàn bộ **OK**, không còn khung nào lệch padding.
