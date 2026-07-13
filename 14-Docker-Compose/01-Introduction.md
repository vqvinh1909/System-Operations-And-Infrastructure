---
title: "01 - Introduction"
module: 14
tags: [docker, sysops-infra, module-14, introduction]
---

# 01 - Giới thiệu Docker Compose

## Tại sao cần Compose — bài toán thực tế

Hãy tưởng tượng Vinh được giao dựng môi trường staging cho một ứng dụng web nội bộ gồm:

- 1 container nginx (reverse proxy, expose port 80)
- 1 container backend Node.js (port 3000, kết nối database)
- 1 container PostgreSQL (lưu dữ liệu, cần volume để không mất data khi container restart)
- 1 container Redis (cache session)

Nếu làm bằng `docker run` thủ công, quy trình sẽ là:

```bash
docker network create app_net
docker volume create db_data

docker run -d --name db --network app_net \
  -e POSTGRES_PASSWORD=secret -v db_data:/var/lib/postgresql/data \
  postgres:16

docker run -d --name redis --network app_net redis:7

# Phải đợi db thật sự sẵn sàng mới chạy backend, nếu không backend crash vì connect fail
docker run -d --name backend --network app_net \
  -e DB_HOST=db -e REDIS_HOST=redis -p 3000:3000 \
  myapp-backend:1.0

docker run -d --name nginx --network app_net \
  -p 80:80 -v ./nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:1.27
```

Đây là 4 lệnh với hơn chục flag. Vấn đề thực tế xảy ra khi:

- **Junior gõ sai tên network** ở lệnh thứ 3 — backend không kết nối được db, mất 30 phút debug mới phát hiện lỗi chính tả.
- **Không ai nhớ đúng thứ tự** — chạy backend trước khi db khởi động xong, ứng dụng crash loop.
- **Máy đồng nghiệp khác chạy lại** — quên volume, dữ liệu mất sau mỗi lần restart.
- **Không review được qua Git** — không ai biết môi trường staging hiện tại thực sự đang chạy image version nào, flag nào, vì lệnh gõ tay không lưu lại đâu cả.

Docker Compose giải quyết toàn bộ vấn đề trên bằng cách gom tất cả khai báo trên vào **một file YAML duy nhất** (`compose.yaml`), commit vào Git, và chạy bằng đúng một lệnh:

```bash
docker compose up -d
```

## Compose là gì, không phải là gì

**Compose LÀ:**
- Một công cụ (CLI plugin của Docker) đọc file YAML khai báo nhiều service, network, volume liên quan đến nhau, rồi tạo/khởi động toàn bộ theo đúng khai báo đó.
- Công cụ dành cho **một máy Docker Engine duy nhất** (single host) — dùng tốt cho dev, staging, demo, hoặc production quy mô nhỏ chạy trên 1 server.
- Một lớp trừu tượng mỏng phía trên Docker Engine API — Compose không tự chạy container, nó ra lệnh cho Docker Engine chạy container giúp nó.

**Compose KHÔNG PHẢI LÀ:**
- Không phải orchestrator đa node — Compose không tự động rải container ra nhiều server, không tự phục hồi (self-healing) khi cả server chết, không có concept "cluster". Đó là việc của Kubernetes hoặc Docker Swarm (sẽ học ở các module sau).
- Không phải công cụ build CI/CD, dù thường được dùng trong pipeline CI để dựng môi trường test.
- Không thay thế `docker run` hoàn toàn — với 1 container đơn giản dùng tạm, `docker run` vẫn nhanh hơn; Compose phát huy giá trị khi có từ 2 service liên kết trở lên.

## Vị trí của Compose trong hệ sinh thái container

```
docker run          -> chạy 1 container, ra lệnh trực tiếp (imperative)
docker compose      -> khai báo nhiều container liên kết, trên 1 host (declarative, single-node)
docker swarm        -> orchestration nhiều node, dùng cú pháp gần giống compose
kubernetes          -> orchestration nhiều node, hệ sinh thái riêng, phức tạp hơn nhiều
```

Compose chính là bước tập tư duy "khai báo trạng thái mong muốn" (declarative) — thay vì viết từng lệnh "làm gì" (imperative), Vinh viết "tôi muốn hệ thống trông như thế nào" và để công cụ tự tính toán cách đạt được trạng thái đó. Đây là tư duy y hệt Ansible playbook (đang học song song) và là tư duy nền tảng của Kubernetes manifest sau này.

## Mục tiêu học tập của module

Sau khi hoàn thành module này, Vinh có thể:

1. Giải thích được Compose Specification là gì, vì sao khóa `version:` không còn dùng.
2. Viết file `compose.yaml` đúng chuẩn 2026: `services`, `networks`, `volumes`, `secrets`, `configs`, `profiles`.
3. Cấu hình `depends_on` với `condition: service_healthy` để đảm bảo thứ tự khởi động thực sự đúng (không chỉ đúng thứ tự container start, mà đúng thứ tự sẵn sàng phục vụ).
4. Phân biệt network mặc định Compose tự tạo, custom network, và external network.
5. Dùng named volume và bind mount đúng ngữ cảnh trong Compose.
6. Dùng `profiles` để bật/tắt nhóm service theo tình huống (dev/debug/prod).
7. Quản lý biến môi trường qua `.env`, hiểu đúng thứ tự ưu tiên.
8. Quản lý secret bằng file-based secrets, không hardcode password vào compose.yaml.
9. Viết `healthcheck` đúng cú pháp và liên hệ với `depends_on`.
10. Debug được các lỗi Compose thường gặp trong môi trường thực tế.

## Liên hệ Ansible (đang học song song)

Vinh sẽ nhận ra nhiều điểm tương đồng:

| Ansible | Docker Compose |
|---|---|
| Playbook (YAML) khai báo trạng thái server mong muốn | compose.yaml khai báo trạng thái container mong muốn |
| `ansible-playbook site.yml` | `docker compose up` |
| Idempotent — chạy lại không phá hỏng gì nếu đã đúng trạng thái | `docker compose up` chạy lại chỉ tạo lại phần thay đổi |
| Variables, `group_vars`, `host_vars` | `.env`, `environment` |
| Vault để giấu secret | Docker secrets (file-based) |

Hiểu rõ Compose sẽ giúp Vinh học Kubernetes sau này nhanh hơn nhiều, vì cấu trúc tư duy "services + networks + volumes khai báo trong YAML" gần như là bản thu nhỏ của Kubernetes Pod/Service/PersistentVolume.

## Tiếp theo

Sang [[02-Theory]] để đi sâu vào cấu trúc Compose Specification và cách Compose vận hành ở tầng internal.
