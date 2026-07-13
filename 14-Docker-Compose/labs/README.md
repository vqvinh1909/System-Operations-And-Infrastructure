---
title: "Labs - Module 14"
module: 14
tags: [docker, sysops-infra, module-14, labs]
---

# Labs - Module 14: Docker Compose

Thư mục lưu compose.yaml, `.env`, secrets mẫu (không chứa giá trị thật) từ [[../05-Labs|05-Labs.md]].

## Môi trường

- VM Linux đã cài Docker Engine 29.x kèm Compose plugin nhánh 5.x.
- Đủ dung lượng để pull các image `nginx`, `postgres`, `redis`, `adminer`.

## File dự kiến sinh ra trong thư mục này

| Thư mục | Sinh ra ở Lab | Nội dung |
|---|---|---|
| `lab14-basic/` | Lab 1 | compose.yaml đầu tiên chuyển đổi từ `docker run` thủ công |
| `lab14-healthcheck/` | Lab 2 | So sánh short syntax vs `condition: service_healthy` |
| `lab14-network/` | Lab 3 | Network tách biệt frontend/backend, bằng chứng cô lập |
| `lab14-env/` | Lab 4 | `.env`, bằng chứng thứ tự ưu tiên biến |
| `lab14-secrets/` | Lab 5 | Secrets file-based, bằng chứng không lộ qua `docker inspect` |
| `lab14-profiles/` | Lab 6 | Profiles debug, ghi chú CI/CD |
| `lab14-final-stack/` | Lab 7 (mini project) | Stack hoàn chỉnh production-grade |

## Lưu ý an toàn

- Thư mục `secrets/` trong mọi lab phải có `.gitignore` loại trừ nội dung thật (`*.txt`), chỉ giữ lại file `.example` mẫu nếu cần minh họa cấu trúc.
- Không commit file `.env` chứa giá trị thật vào Git — chỉ commit `.env.example`.
