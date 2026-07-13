---
title: "08 - Summary"
module: 10
tags: [docker, sysops-infra, module-10, summary]
---

# 08 - Summary: Docker Images

## Những gì đã học

- **Mỗi instruction Dockerfile tạo một layer** (trừ metadata-only như `ENV`/`LABEL`/`CMD`), xếp chồng theo OverlayFS (Module 08). Cache key phụ thuộc instruction + nội dung file COPY + cache key layer cha — miss ở đâu, miss theo domino từ đó trở đi.
- **Nguyên tắc sắp xếp Dockerfile**: ít đổi trước (FROM, cài dependency theo manifest), đổi thường xuyên sau cùng (COPY source code).
- **ENTRYPOINT vs CMD**: ENTRYPOINT cố định, CMD là tham số mặc định có thể ghi đè. Exec form bắt buộc để ứng dụng làm đúng PID 1, nhận SIGTERM đúng cách.
- **ARG vs ENV**: ARG chỉ tồn tại lúc build, ENV tồn tại cả runtime. Không bao giờ dùng cả hai để truyền secret — dùng BuildKit secret mount.
- **USER non-root** là best practice bảo mật bắt buộc — giảm mức độ nghiêm trọng nếu container bị escape.
- **Multi-stage build**: tách stage builder (toolchain nặng) khỏi stage final (chỉ artifact cần thiết) — giảm kích thước và bề mặt tấn công đáng kể.
- **BuildKit**: builder mặc định từ Docker Engine 23.0, hỗ trợ cache mount (cache dependency xuyên nhiều lần build, không vào layer) và secret mount (secret không lưu vào layer), cùng buildx cho build đa nền tảng.
- **.dockerignore**: loại trừ file khỏi build context trước khi gửi lên daemon — lớp bảo vệ đầu tiên chống leak secret.

## Checklist tự đánh giá

- [ ] Viết được Dockerfile đúng thứ tự tối ưu cache cho một ứng dụng thực tế.
- [ ] Giải thích đúng 4 tổ hợp hành vi ENTRYPOINT/CMD.
- [ ] Viết được Dockerfile multi-stage giảm kích thước image ít nhất 5-10 lần so với single-stage.
- [ ] Dùng đúng cache mount và secret mount của BuildKit, giải thích khác biệt với layer cache.
- [ ] Tự rà soát được một Dockerfile có đủ: USER non-root, pin version, .dockerignore, HEALTHCHECK.
- [ ] Chẩn đoán được nguyên nhân image phình to hoặc cache không hit bằng `docker history`.

## Liên kết tới Module tiếp theo

Module này tập trung vào **xây dựng** image. [[../11-Container-Runtime/README|Module 11 - Container Runtime]] quay lại phía **chạy** container từ image đó: vòng đời chi tiết hơn, giới hạn tài nguyên (`--memory`/`--cpus` áp dụng thực tế lên image bạn vừa build), restart policy, và các lệnh vận hành hàng ngày (`docker events`, `docker stats`, `docker system df`, `docker system prune`) — những công cụ bạn sẽ dùng liên tục để quản lý image/container đã tích lũy qua quá trình build lặp đi lặp lại ở module này.

> [!tip] Ghi nhớ cốt lõi
> Một Dockerfile tốt không chỉ "build ra được image chạy đúng" — nó phải tối ưu cache (tốc độ CI/CD), tối thiểu kích thước (băng thông, chi phí lưu trữ), và tối thiểu quyền hạn (bảo mật). Ba tiêu chí này luôn đi cùng nhau trong một Dockerfile viết đúng chuẩn production.
