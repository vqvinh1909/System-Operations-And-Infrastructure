---
title: "Labs - Module 10"
module: 10
tags: [docker, sysops-infra, module-10, labs]
---

# Labs - Module 10: Docker Images

Thư mục lưu Dockerfile, source code demo, và ghi chú thực nghiệm từ [[../05-Labs|05-Labs.md]].

## Môi trường

- Docker Engine 29.x, BuildKit là builder mặc định (không cần bật thủ công).
- Truy cập internet để pull base image (`python`, `node`, `golang`, `alpine`) từ Docker Hub.

## File dự kiến sinh ra trong thư mục này

| File/Thư mục | Sinh ra ở Lab | Nội dung |
|---|---|---|
| `lab10-basic/` | Lab 1 | Dockerfile + app.py, ghi chú layer/cache quan sát được |
| `lab10-cache/` | Lab 2, Lab 5 | `Dockerfile.bad`, `Dockerfile.good`, bảng so sánh thời gian build |
| `lab10-entrypoint/` | Lab 3 | Dockerfile ENTRYPOINT+CMD+HEALTHCHECK, server.py |
| `lab10-multistage/` | Lab 4 | `Dockerfile.single`, `Dockerfile.multi`, số liệu kích thước image đo được |
| `<ten-service>/` | Lab 6 (mini project) | Dockerfile production-grade hoàn chỉnh, `.dockerignore`, source code, ghi chú thiết kế |

## Lưu ý

- Ghi lại số liệu thật đo được trên máy bạn (thời gian build, kích thước image) — không suy đoán hay copy số liệu mẫu trong tài liệu.
- Dọn dẹp image/container thử nghiệm sau mỗi lab (`docker rmi`, `docker builder prune`) để không tích tụ rác trên VM lab.
