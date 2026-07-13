---
title: "08 - Summary"
module: 18
tags: [docker, sysops-infra, module-18, security, summary]
---

# 08 — Tóm tắt Module 18

## Những gì bạn đã học

- **Docker rootful** chạy daemon bằng root, khiến "quyền chạy lệnh docker" tương đương "quyền root trên host" — một rủi ro kiến trúc, không chỉ là chi tiết cấu hình.
- **Rootless Docker** chạy daemon bằng user thường, dùng user namespace + RootlessKit + slirp4netns, giảm thiểu triệt để hậu quả nếu container bị chiếm (breakout), đánh đổi một phần hiệu năng network và một số tính năng cần quyền root thật.
- **Capabilities**: quyền root bị chia nhỏ thành ~40 mảnh độc lập; thực hành chuẩn là `--cap-drop=ALL` rồi chỉ `--cap-add` đúng thứ ứng dụng thật sự cần.
- **Seccomp** chặn ở tầng syscall, **AppArmor** chặn ở tầng truy cập tài nguyên (file/network/capability) — hai cơ chế bổ sung nhau, nên dùng đồng thời ở chế độ mặc định (enforcing), liên hệ trực tiếp tới Security Hardening đã học ở Linux Sysadmin.
- **Image scanning** (Trivy, Docker Scout) cần trở thành một **gate bắt buộc** trong CI/CD, không phải bước tham khảo — và cần chạy lại định kỳ, không chỉ một lần lúc build.
- **Docker Bench for Security** tự động chấm điểm hệ thống theo CIS Docker Benchmark — vẫn được duy trì năm 2026, nhưng nên clone repo và tự build thay vì dùng image dựng sẵn đã lỗi thời.
- Toàn bộ 5 kỹ thuật trên hợp thành mô hình **Defense in Depth**: không lớp nào hoàn hảo riêng lẻ, nhưng kết hợp lại tạo ra nhiều rào cản độc lập mà kẻ tấn công phải vượt qua đồng thời.

## Checklist hardening Docker tối thiểu

| Hạng mục | Hành động |
|---|---|
| Daemon | Cân nhắc Rootless Docker cho môi trường phù hợp; nếu vẫn rootful, giới hạn nghiêm ngặt ai được vào nhóm `docker` |
| Container | `--cap-drop=ALL` mặc định, chỉ `--cap-add` khi có lý do rõ ràng |
| Kernel | Giữ nguyên seccomp profile mặc định, không dùng `unconfined` trừ khi debug tạm thời trên lab |
| Tài nguyên | Giữ nguyên AppArmor `docker-default` (hoặc SELinux tương đương), không tắt |
| Image | Scan bắt buộc trước khi push, gate CI/CD theo mức độ nghiêm trọng CVE |
| Vận hành | Chạy Docker Bench for Security định kỳ, theo dõi xu hướng `[WARN]` qua thời gian |

## Cầu nối sang Module 19

Bạn đã học cách quan sát container (Module 17) và bảo vệ container (Module 18). Nhưng cả hai kỹ năng đó đều giả định một điều: **bạn biết chuyện gì đang xảy ra khi có sự cố**. Thực tế, phần lớn thời gian của một sysadmin/DevOps không phải lúc mọi thứ chạy êm — mà là lúc một thứ gì đó *không* hoạt động và bạn phải tìm ra vì sao trong áp lực thời gian. [[../19-Troubleshooting/README|Module 19 — Troubleshooting]] tổng hợp và đào sâu kỹ năng chẩn đoán sự cố container ở bốn tầng: Container, Network, Storage, Performance — vượt xa những gì các mục "Troubleshooting" nhỏ ở từng module trước đã đề cập.
