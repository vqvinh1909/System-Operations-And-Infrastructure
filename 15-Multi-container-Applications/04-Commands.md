---
title: "04 - Commands"
module: 15
tags: [docker, sysops-infra, module-15, commands, reference, compose]
---

# 04 - Commands: compose.yaml hoàn chỉnh, lệnh vận hành stack

## 1. `compose.yaml` — stack 6 service hoàn chỉnh

```yaml
name: shopstack

services:
  nginx:
    image: nginx:1.27-alpine
    ports:
      - "${HTTP_PORT:-80}:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      backend:
        condition: service_started
    networks: [frontend]
    restart: unless-stopped

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: shopstack-backend:1.0
    environment:
      DB_HOST: db
      DB_NAME: shopdb
      REDIS_HOST: redis
      RABBITMQ_HOST: rabbitmq
      DB_PASSWORD_FILE: /run/secrets/db_password
    env_file:
      - .env.backend
    secrets:
      - db_password
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks: [frontend, backend]
    restart: unless-stopped

  worker:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    image: shopstack-worker:1.0
    environment:
      RABBITMQ_HOST: rabbitmq
      DB_HOST: db
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    depends_on:
      rabbitmq:
        condition: service_healthy
      db:
        condition: service_healthy
    networks: [backend]
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: shopdb
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d shopdb"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 15s
    networks: [backend]
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks: [backend]
    restart: unless-stopped

  rabbitmq:
    image: rabbitmq:4-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: shopapp
      RABBITMQ_DEFAULT_PASS_FILE: /run/secrets/rabbitmq_password
    secrets:
      - rabbitmq_password
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s
    ports:
      - "15672:15672"
    profiles: ["debug"]
    networks: [backend]
    restart: unless-stopped

networks:
  frontend:
  backend:

volumes:
  db_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
  rabbitmq_password:
    file: ./secrets/rabbitmq_password.txt
```

**Điểm cần chú ý**: `rabbitmq` publish port `15672` (management UI) nhưng gắn `profiles: ["debug"]` — mặc định UI quản trị **không** lộ ra ngoài trên production, chỉ bật khi cần debug tường minh. `worker` không nằm trong `frontend` — nó chỉ tiêu thụ message từ `rabbitmq`, không bao giờ nhận request trực tiếp từ Nginx.

## 2. Vận hành stack

```bash
# Dung toan bo stack (khong co rabbitmq management UI, vi co profiles debug)
docker compose up -d

# Dung ca cong cu debug (rabbitmq management UI)
docker compose --profile debug up -d

# Xem trang thai tung service, cot HEALTH phai la "healthy" truoc khi bao ho thanh cong hoan toan
docker compose ps

# Xem log tong hop toan bo stack, theo doi realtime
docker compose logs -f

# Chi xem log 1 thanh phan
docker compose logs -f worker
```

## 3. Kiểm tra sức khỏe từng thành phần thủ công

```bash
# Kiem tra database accept connection
docker compose exec db pg_isready -U postgres -d shopdb

# Kiem tra Redis phan hoi
docker compose exec redis redis-cli ping

# Kiem tra RabbitMQ (can profile debug de co container nay chay)
docker compose --profile debug exec rabbitmq rabbitmq-diagnostics ping

# Xem so luong message dang cho trong queue qua management UI (can profile debug)
curl -u shopapp:<password> http://localhost:15672/api/queues
```

## 4. Rolling update một thành phần mà không downtime toàn stack

```bash
# Chi build lai va thay the "backend", cac service khac khong bi dung
docker compose up -d --build backend

# Xac nhan cac service khac khong bi restart theo
docker compose ps
```

## 5. Backup dữ liệu database trong stack (kết hợp Module 13)

```bash
docker run --rm \
  -v shopstack_db_data:/source:ro \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/shopdb-$(date +%Y%m%d-%H%M).tar.gz -C /source .
```

Lưu ý tên volume thật đã được tiền tố bởi project name (`shopstack_db_data`), đúng nguyên tắc đặt tên đã học ở Module 14.

## 6. Dừng an toàn toàn bộ stack

```bash
# Dung, giu nguyen network + volume (khoi phuc nhanh sau nay)
docker compose stop

# Dung va xoa container + network, GIU volume (an toan cho du lieu)
docker compose down

# CHI dung khi chac chan khong can du lieu nua - xoa ca volume
docker compose down -v
```

Tiếp theo: [[05-Labs]] để thực hành dựng toàn bộ stack này từng bước.
