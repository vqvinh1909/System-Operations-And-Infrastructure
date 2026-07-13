---
title: "04 - Commands"
module: 14
tags: [docker, sysops-infra, module-14, commands, reference]
---

# 04 - Commands: docker compose CLI, compose.yaml mẫu đầy đủ

## 1. Lệnh cơ bản

```bash
docker compose up -d                 # dung nen, chay tat ca service (khong co profiles) o che do nen
docker compose up -d --build         # buoc build lai image truoc khi start (voi service co "build:")
docker compose ps                    # xem trang thai tung service/container trong project
docker compose logs -f               # xem log tat ca service, theo doi realtime
docker compose logs -f backend       # chi xem log 1 service cu the
docker compose down                  # dung va xoa container + network (KHONG xoa named volume)
docker compose down -v               # xoa ca named volume - CAN THAN, mat du lieu
docker compose stop                  # chi dung container, khong xoa gi ca
docker compose restart backend       # restart 1 service cu the
```

## 2. Kiểm tra và debug cấu hình

```bash
# In ra cau hinh cuoi cung sau khi merge nhieu file + noi suy bien - rat huu ich khi debug
docker compose config

# Chi validate, khong in ra
docker compose config --quiet

# Chay voi profile cu the
docker compose --profile debug up -d

# Chi build, khong start
docker compose build

# Buoc tao lai container ke ca khi Compose nghi la khong doi gi (xem 06-Troubleshooting)
docker compose up -d --force-recreate
```

## 3. Chạy lệnh một lần trong service (không ảnh hưởng container chính)

```bash
# Chay mot lenh tam thoi trong 1 container moi cua service do (vd chay migration)
docker compose run --rm backend python manage.py migrate

# Exec vao container dang chay cua 1 service
docker compose exec backend sh
```

## 4. File `compose.yaml` mẫu đầy đủ

```yaml
name: myshop

services:
  nginx:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      backend:
        condition: service_started
    networks: [frontend]
    restart: unless-stopped

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: myapp-backend:1.0
    environment:
      DB_HOST: db
      REDIS_HOST: redis
      DB_PASSWORD_FILE: /run/secrets/db_password
    env_file:
      - .env.backend
    secrets:
      - db_password
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks: [frontend, backend]
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: myshop
    volumes:
      - db_data:/var/lib/postgresql/data
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s
    networks: [backend]
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    networks: [backend]
    restart: unless-stopped

  adminer:
    image: adminer
    ports:
      - "8081:8080"
    profiles: ["debug"]
    networks: [backend]

networks:
  frontend:
  backend:

volumes:
  db_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**Điểm cần chú ý trong file mẫu**: `db` không nằm trong network `frontend` — không thể bị truy cập trực tiếp từ `nginx` hay bên ngoài, chỉ `backend` thấy được `db`. `adminer` có `profiles: ["debug"]` nên mặc định không chạy khi `docker compose up -d`, chỉ chạy khi thêm `--profile debug`. Password không hề xuất hiện dạng chữ rõ (plaintext) trong file YAML — chỉ tham chiếu tới `secrets.db_password.file`, nội dung thật nằm ở `./secrets/db_password.txt` (không commit vào Git).

## 5. `.env` mẫu và cách dùng

```bash
# File .env cung thu muc voi compose.yaml
# COMPOSE_PROJECT_NAME=myshop-staging
NGINX_PORT=80
BACKEND_IMAGE_TAG=1.0
```

```yaml
services:
  nginx:
    ports:
      - "${NGINX_PORT}:80"
  backend:
    image: myapp-backend:${BACKEND_IMAGE_TAG}
```

```bash
# Chi dinh file .env khac (vd .env.production) thay vi mac dinh
docker compose --env-file .env.production up -d
```

Tiếp theo: [[05-Labs]] để thực hành trực tiếp toàn bộ cú pháp trên.
