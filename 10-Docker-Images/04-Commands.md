---
title: "04 - Commands"
module: 10
tags: [docker, sysops-infra, module-10, commands, dockerfile-examples]
---

# 04 - Commands: docker build, Build Args, Tagging, Dockerfile mẫu

## 1. Lệnh `docker build` cơ bản

```bash
# Build từ Dockerfile trong thư mục hiện tại, gán tag
docker build -t myapp:1.4.2 .

# Chỉ định Dockerfile khác tên/vị trí mặc định
docker build -f docker/Dockerfile.prod -t myapp:1.4.2 .

# Build không dùng cache (buộc chạy lại toàn bộ)
docker build --no-cache -t myapp:1.4.2 .

# Build và gán nhiều tag cùng lúc
docker build -t myapp:1.4.2 -t myapp:latest -t registry.company.local/myapp:1.4.2 .
```

Dấu `.` cuối lệnh chính là **build context** — thư mục được gửi lên BuildKit, cũng là "gốc" cho mọi đường dẫn tương đối trong `COPY`/`ADD`. Đây là lỗi rất hay gặp: nhầm build context với vị trí Dockerfile (xem [[06-Troubleshooting]] mục "COPY sai context").

## 2. Build Args — truyền tham số lúc build

```bash
docker build \
  --build-arg NODE_VERSION=20.15 \
  --build-arg BUILD_ENV=production \
  -t myapp:1.4.2 .
```

Tương ứng trong Dockerfile:

```dockerfile
ARG NODE_VERSION=20.15
FROM node:${NODE_VERSION}-alpine

ARG BUILD_ENV=development
ENV NODE_ENV=${BUILD_ENV}
```

Nếu không truyền `--build-arg`, Docker dùng giá trị mặc định khai báo sau `ARG` (nếu có). Nhắc lại nguyên tắc bảo mật: **không dùng build-arg để truyền secret** — dùng `--secret` (mục 5 dưới đây).

## 3. Tagging — quy ước đặt tên image

```bash
# Cú pháp đầy đủ: [registry-host[:port]/]repository[:tag]
docker build -t registry.company.local:5000/team-billing/api-service:1.4.2 .

# Gán thêm tag "latest" cho cùng image vừa build (chỉ nên dùng nội bộ, không dùng cho production pin version)
docker tag registry.company.local:5000/team-billing/api-service:1.4.2 \
           registry.company.local:5000/team-billing/api-service:latest
```

**Best practice tagging trong doanh nghiệp:** dùng semantic version (`1.4.2`) hoặc Git commit SHA (`sha-a1b2c3d`) làm tag chính để **truy vết chính xác** image nào tương ứng commit nào. Tag `latest` chỉ nên dùng làm "con trỏ tiện lợi" cho môi trường dev, **không bao giờ deploy production dựa trên tag `latest`** — vì `latest` không đảm bảo tính bất biến (ai đó build lại `latest` là nội dung image thay đổi hoàn toàn dù tên không đổi).

## 4. Dockerfile mẫu đầy đủ — ứng dụng Node.js, đơn stage (baseline)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20.15-alpine

LABEL maintainer="devops@company.com" \
      description="Internal billing API service"

# Tạo user non-root trước
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy dependency manifest trước để tận dụng cache
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy source code SAU CÙNG vì đây là phần đổi thường xuyên nhất
COPY --chown=appuser:appgroup . .

ENV NODE_ENV=production
EXPOSE 8080

USER appuser

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

ENTRYPOINT ["node"]
CMD ["server.js"]
```

Dockerfile này minh họa gần như đầy đủ instruction cơ bản: `FROM` pin version, `LABEL`, `USER` non-root, thứ tự `COPY` tối ưu cache, `ENV`, `EXPOSE` (chỉ tài liệu), `HEALTHCHECK`, và cách kết hợp `ENTRYPOINT`+`CMD`.

## 5. Dockerfile multi-stage build thực tế — Go service

```dockerfile
# syntax=docker/dockerfile:1

# ---------- Stage 1: builder ----------
FROM golang:1.23 AS builder

WORKDIR /src

# Copy go.mod/go.sum trước để cache "go mod download" theo dependency
COPY go.mod go.sum ./

# Cache mount cho Go module cache — không lưu vào layer, tồn tại xuyên nhiều lần build
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

# Cache mount thêm cho go build cache, và secret mount để git clone private module nếu cần
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/server .

# ---------- Stage 2: final ----------
FROM alpine:3.20

RUN apk add --no-cache ca-certificates \
    && addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder /out/server /app/server

USER appuser
WORKDIR /app

EXPOSE 8080
STOPSIGNAL SIGTERM

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD /app/server -healthcheck || exit 1

ENTRYPOINT ["/app/server"]
```

**Điểm cần chú ý trong ví dụ này:**

- `# syntax=docker/dockerfile:1` ở dòng đầu tiên là **bắt buộc** để dùng cú pháp `--mount` của BuildKit — thiếu dòng này, `--mount` sẽ bị hiểu nhầm thành một phần của lệnh shell, build lỗi hoặc sai kết quả.
- `--mount=type=cache,target=/go/pkg/mod`: cache Go module giữa các lần build — lần build sau không cần tải lại dependency đã tải trước đó, dù layer `COPY go.mod go.sum` có bị cache miss.
- `CGO_ENABLED=0`: compile static binary, không phụ thuộc glibc/libc của base image — đây là điều kiện để có thể dùng `FROM scratch` hoặc `alpine` tối giản ở stage final.
- `ca-certificates` được cài ở stage final vì service Go cần verify TLS certificate khi gọi API HTTPS ra ngoài — đây là phụ thuộc runtime, khác với phụ thuộc build-time.
- Kết quả: image final chỉ có Alpine base + `ca-certificates` + 1 binary, không có Go compiler, không có source code.

## 6. Cache mount cho các package manager phổ biến khác

```dockerfile
# syntax=docker/dockerfile:1

# APT (Debian/Ubuntu) — sharing=locked tránh race condition khi build song song
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends curl

# pip (Python)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# npm (Node.js)
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Maven (Java)
RUN --mount=type=cache,target=/root/.m2/repository \
    mvn -B package
```

**Vì sao cache mount khác với layer cache thông thường:** layer cache (mục [[02-Theory]]) chỉ HIT khi input không đổi. Cache mount thì **luôn khả dụng** ở mọi lần build (kể cả khi source code đổi liên tục), vì nó không nằm trong layer của image — nó là một vùng lưu trữ riêng do BuildKit quản lý, tồn tại độc lập giữa các lần build.

## 7. Secret mount — truyền secret an toàn lúc build

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.20

# Dùng token trong RUN nhưng KHÔNG lưu vào layer/image
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) \
    npm config set //registry.npmjs.org/:_authToken=$NPM_TOKEN
```

```bash
docker build --secret id=npm_token,src=./npm_token.txt -t myapp:1.4.2 .
```

Secret chỉ tồn tại trong bộ nhớ tạm (`/run/secrets/npm_token`) trong đúng RUN đó, **không bao giờ được ghi vào bất kỳ layer nào** — khác hẳn `ARG`/`ENV`, vốn để lại dấu vết vĩnh viễn trong `docker history`.

## 8. buildx — build đa nền tảng

```bash
# Tạo builder instance mới (chỉ cần 1 lần)
docker buildx create --name multiarch-builder --use

# Build đồng thời cho amd64 và arm64, push thẳng lên registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t registry.company.local/team-billing/api-service:1.4.2 \
  --push .
```

Dùng khi hạ tầng doanh nghiệp có cả server x86 (`amd64`) và server ARM (ví dụ AWS Graviton, `arm64`) — build 1 lần, ra manifest đa kiến trúc, Docker daemon trên từng loại host tự kéo đúng phiên bản phù hợp.

## 9. .dockerignore mẫu

```
.git
.gitignore
node_modules
npm-debug.log
Dockerfile*
.dockerignore
README.md
.env
.env.*
*.local
coverage/
.vscode/
.idea/
```

`.dockerignore` được đọc **trước khi** build context được gửi lên daemon — file khớp pattern sẽ **không bao giờ** rời khỏi máy bạn, kể cả khi Dockerfile lỡ có `COPY . .`. Đây là lớp bảo vệ quan trọng chống leak file `.env` chứa secret vào image.

## 10. Các lệnh liên quan khác

```bash
# Xem lịch sử layer của 1 image (kể cả build-arg từng dùng — nhắc lại cảnh báo bảo mật)
docker history myapp:1.4.2

# Xem chi tiết kích thước từng layer
docker history --no-trunc myapp:1.4.2

# Kiểm tra image có bao nhiêu layer, kích thước tổng
docker inspect myapp:1.4.2

# Dọn build cache không dùng nữa (giải phóng /var/lib/docker)
docker builder prune

# Dọn toàn bộ build cache, kể cả cache đang dùng gần đây
docker builder prune --all --force
```

Tiếp theo: [[05-Labs]] để thực hành trực tiếp các Dockerfile và lệnh trên.
