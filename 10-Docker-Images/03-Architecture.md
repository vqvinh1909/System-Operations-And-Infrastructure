---
title: "03 - Architecture"
module: 10
tags: [docker, sysops-infra, module-10, architecture, diagram]
---

# 03 - Architecture: Sơ đồ Layer Cache và Multi-stage Build

Tất cả sơ đồ dưới đây vẽ bằng ASCII (không dùng Mermaid), đã được đo bằng script Python để đảm bảo mọi khung chữ nhật căn đều (border trên/dưới và các dòng nội dung có độ dài bằng nhau).

## 1. Luồng tổng quát: từ `docker build` đến OCI Image

Sơ đồ này cho thấy build context được gửi lên BuildKit ra sao, và BuildKit tạo ra image tuân theo OCI Image Spec như thế nào (nhắc lại: OCI Image Spec hiện tại v1.1.0).

```
+-----------------+       +-----------------------+       +---------------------+       +---------------------+
| docker build .  |       | Build Context         |       | BuildKit            |       | OCI Image           |
| (Docker CLI /   |       | (thu muc gui len      |       | (buildkitd)         |       | (cac layer +        |
|  buildx client) |       |  daemon, tru file     |       | giai Dockerfile     |       |  manifest + config, |
|                 |  -->  |  trong .dockerignore) |  -->  | thanh DAG cac buoc, |  -->  |  luu trong local    |
|                 |       |                       |       | chay song song,     |       |  image store hoac   |
|                 |       |                       |       | cache theo content  |       |  push len registry) |
|                 |       |                       |       | hash                |       |                     |
+-----------------+       +-----------------------+       +---------------------+       +---------------------+
```

**Diễn giải từng bước:**

1. **Docker CLI / buildx client**: bạn chạy `docker build .` hoặc `docker buildx build .`.
2. **Build Context**: thư mục `.` được đóng gói gửi lên, nhưng **loại trừ mọi thứ khớp `.dockerignore`** — đây là lý do file `.dockerignore` ảnh hưởng trực tiếp tới tốc độ và bảo mật build (xem [[02-Theory]] mục 19, [[06-Troubleshooting]]).
3. **BuildKit (buildkitd)**: đọc Dockerfile, dựng DAG các bước, xác định bước nào cache HIT (tái sử dụng), bước nào phải chạy thật.
4. **OCI Image**: kết quả cuối — tập hợp layer (mỗi layer là 1 tar diff), file manifest, và file config (chứa ENTRYPOINT, ENV, CMD...) — đúng chuẩn OCI Image Spec để bất kỳ runtime OCI-compliant nào (runc, containerd) cũng chạy được.

## 2. Layer Cache Stack — mỗi instruction là 1 layer, cache hit/miss theo thứ tự

Ví dụ minh họa build một Dockerfile Node.js. Layer được đánh số từ dưới lên (Layer 1 = `FROM`, layer thấp nhất, layer sau xếp chồng lên trên). Giả định lần build này chỉ sửa code trong thư mục `src/`, không đổi `package.json`:

```
+---------+---------------------------------+-----------------------------+
| Layer   | Dockerfile Instruction          | Cache Status                |
+---------+---------------------------------+-----------------------------+
| Layer 8 | RUN chmod +x /app/entrypoint.sh | MISS (COPY o tren thay doi) |
| Layer 7 | COPY . /app                     | MISS (source code thay doi) |
| Layer 6 | RUN npm ci                      | HIT (cache, deps chua doi)  |
| Layer 5 | COPY package*.json /app/        | HIT (checksum khong doi)    |
| Layer 4 | WORKDIR /app                    | HIT                         |
| Layer 3 | RUN apk add --no-cache curl     | HIT                         |
| Layer 2 | ENV NODE_ENV=production         | HIT                         |
| Layer 1 | FROM node:20-alpine             | HIT (base image)            |
+---------+---------------------------------+-----------------------------+
```

**Đọc sơ đồ này như thế nào:**

- Layer 1-6 **HIT** vì `FROM`, `ENV`, `RUN apk add`, `WORKDIR`, và cặp `COPY package*.json` + `RUN npm ci` không phụ thuộc vào source code trong `src/` — chúng chỉ phụ thuộc `package.json`/`package-lock.json`, mà file này không đổi trong lần build này.
- Layer 7 **MISS** ngay khi `COPY . /app` chạm phải file trong `src/` đã sửa — checksum đổi, cache mất hiệu lực.
- Layer 8 **MISS theo domino**: dù lệnh `RUN chmod +x` không đổi, nó xây trên layer 7 (đã miss), nên **bắt buộc phải chạy lại**, không thể tái sử dụng layer cũ.
- Đây chính là lý do quy tắc "COPY dependency manifest trước, COPY toàn bộ source code sau cùng" (mục 16, [[02-Theory]]) giúp tiết kiệm hàng phút build mỗi khi chỉ sửa code, không đụng dependency.

## 3. Multi-stage Build — builder stage vs final stage

Ví dụ build một service Go: stage 1 (`builder`) chứa toàn bộ toolchain Go để compile, stage 2 (`final`) chỉ lấy đúng 1 binary đã compile, bỏ lại mọi thứ khác.

```
+-------------------------------+        +------------------------------+
| STAGE 1: builder              |        | STAGE 2: final               |
| FROM golang:1.23 AS builder   |        | FROM alpine:3.20             |
|                               |        |                              |
| WORKDIR /src                  |        | RUN adduser -D appuser       |
| COPY go.mod go.sum ./         |        | COPY --from=builder \        |
| RUN go mod download           |        |      /out/server /app/server |
| COPY . .                      |  ===>  | USER appuser                 |
| RUN go build -o /out/server . |        | ENTRYPOINT ["/app/server"]   |
|                               |        |                              |
| Image tam thoi ~900MB         |        | Image final ~15MB            |
| (chua toolchain Go, source,   |        | (chi co binary +             |
|  cache, build tools)          |        |  alpine base)                |
+-------------------------------+        +------------------------------+
```

**Diễn giải:**

- **Stage 1 (builder)** dùng image `golang:1.23` đầy đủ (chứa compiler, standard library, toolchain) — image này có thể nặng gần 1GB, nhưng **không bao giờ được đẩy lên registry hay chạy production**. Nó chỉ tồn tại tạm thời trong quá trình build, bị BuildKit giữ lại trong cache để lần build sau tái sử dụng (nếu `go.mod`/`go.sum` không đổi).
- Mũi tên `===>` biểu diễn instruction `COPY --from=builder /out/server /app/server` — **chỉ 1 file duy nhất** (binary đã compile) được mang sang stage 2, không mang theo source code, không mang theo Go module cache, không mang theo compiler.
- **Stage 2 (final)** dùng `alpine:3.20` tối giản, chỉ có binary + user non-root — đây mới là image thực sự được tag, push lên registry, và chạy trong production.
- Kết quả: image production nhỏ hơn image build **hàng chục lần**, đồng thời loại bỏ hoàn toàn source code và toolchain khỏi image chạy thật — giảm rủi ro leak mã nguồn và giảm CVE bề mặt tấn công.

Xem cú pháp Dockerfile multi-stage đầy đủ (kèm cache mount) tại [[04-Commands]].
