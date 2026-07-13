---
title: "05 - Labs"
module: 10
tags: [docker, sysops-infra, module-10, labs, hands-on]
---

# 05 - Labs: Docker Images thực hành

Yêu cầu trước khi bắt đầu: đã cài Docker Engine (29.x), có quyền chạy `docker` (không cần `sudo` nếu user đã trong group `docker` — xem lại Module 09 nếu chưa cấu hình). Toàn bộ code/config lab nên lưu tại [[labs/README|thư mục labs/]] của module này.

## Lab 1 (Basic) — Viết Dockerfile đầu tiên, quan sát layer

**Mục tiêu:** hiểu mỗi instruction tạo 1 layer, và cách `docker history` phản ánh điều đó.

**Bước 1.** Tạo thư mục lab và file ứng dụng tối giản:

```bash
mkdir -p ~/lab10-basic && cd ~/lab10-basic
cat > app.py << 'EOF'
print("Hello from Docker image lab")
EOF
```

**Bước 2.** Viết Dockerfile:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
RUN echo "Build hoan tat" > /app/build-note.txt
CMD ["python", "app.py"]
```

**Bước 3.** Build và gán tag:

```bash
docker build -t lab10-basic:1.0 .
```

**Bước 4.** Quan sát layer:

```bash
docker history lab10-basic:1.0
docker history --no-trunc lab10-basic:1.0
```

**Yêu cầu hoàn thành:**
- Đếm số layer tương ứng số instruction (trừ `FROM` là nhiều layer sẵn có từ base image).
- Chạy thử container: `docker run --rm lab10-basic:1.0` — xác nhận in ra đúng dòng chữ.
- Sửa `app.py` (đổi nội dung in ra), build lại, quan sát: layer nào HIT, layer nào MISS (dùng `docker build` — output có ghi `CACHED` cho layer HIT).

## Lab 2 (Basic) — Layer cache: thứ tự COPY ảnh hưởng cache thế nào

**Mục tiêu:** chứng minh bằng thực nghiệm nguyên tắc "COPY dependency trước, source code sau".

**Bước 1.** Tạo project Node.js tối giản:

```bash
mkdir -p ~/lab10-cache && cd ~/lab10-cache
cat > package.json << 'EOF'
{ "name": "lab10-cache", "version": "1.0.0", "dependencies": {} }
EOF
cat > server.js << 'EOF'
console.log("v1: server starting");
EOF
```

**Bước 2.** Viết Dockerfile SAI thứ tự (để quan sát vấn đề):

```dockerfile
# Dockerfile.bad
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
```

**Bước 3.** Build lần đầu, ghi lại thời gian:

```bash
time docker build -f Dockerfile.bad -t lab10-cache:bad .
```

**Bước 4.** Sửa `server.js` (chỉ đổi nội dung log, không đụng `package.json`), build lại:

```bash
sed -i 's/v1/v2/' server.js
time docker build -f Dockerfile.bad -t lab10-cache:bad .
```

**Quan sát:** dù `package.json` không đổi, `RUN npm install` vẫn chạy lại (MISS) vì nó nằm SAU `COPY . .`.

**Bước 5.** Viết Dockerfile ĐÚNG thứ tự:

```dockerfile
# Dockerfile.good
FROM node:20-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
```

**Bước 6.** Lặp lại thử nghiệm sửa `server.js` rồi build lại với `Dockerfile.good`, so sánh thời gian build và output `CACHED`.

**Yêu cầu hoàn thành:** viết lại (ghi chú trong `labs/`) bảng so sánh thời gian build giữa 2 cách, và giải thích bằng ngôn ngữ của bạn tại sao có sự khác biệt.

## Lab 3 (Intermediate) — ENTRYPOINT + CMD, USER non-root, HEALTHCHECK

**Mục tiêu:** thực hành kết hợp ENTRYPOINT/CMD, chạy container non-root, và cấu hình healthcheck hoạt động thật.

**Bước 1.** Viết ứng dụng HTTP tối giản có endpoint `/health`:

```bash
mkdir -p ~/lab10-entrypoint && cd ~/lab10-entrypoint
cat > server.py << 'EOF'
from http.server import BaseHTTPRequestHandler, HTTPServer
import sys

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/health":
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"ok")
        else:
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"hello")

if __name__ == "__main__":
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 8080
    HTTPServer(("0.0.0.0", port), Handler).serve_forever()
EOF
```

**Bước 2.** Viết Dockerfile dùng ENTRYPOINT cố định + CMD làm tham số mặc định (port):

```dockerfile
FROM python:3.12-slim
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
WORKDIR /app
COPY server.py .
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')" || exit 1
ENTRYPOINT ["python", "server.py"]
CMD ["8080"]
```

**Bước 3.** Build và chạy, publish port thật (nhắc lại: EXPOSE không tự publish):

```bash
docker build -t lab10-entrypoint:1.0 .
docker run -d --name lab10-app -p 8080:8080 lab10-entrypoint:1.0
```

**Bước 4.** Xác nhận chạy bằng user non-root:

```bash
docker exec lab10-app whoami
docker exec lab10-app id
```

**Bước 5.** Quan sát healthcheck chuyển trạng thái:

```bash
docker ps   # cột STATUS sẽ hiện "starting" rồi chuyển "healthy"
```

**Bước 6.** Thử ghi đè CMD (đổi port) mà không sửa ENTRYPOINT:

```bash
docker run --rm -p 9090:9090 lab10-entrypoint:1.0 9090
docker exec -it $(docker ps -lq) sh -c "netstat -tlnp 2>/dev/null || ss -tlnp"
```

**Yêu cầu hoàn thành:** giải thích tại sao `docker exec ... whoami` không trả về `root`, và tại sao đổi tham số cuối lệnh `docker run` lại đổi được port mà không cần sửa Dockerfile.

## Lab 4 (Intermediate) — Multi-stage build giảm kích thước image

**Mục tiêu:** đo lường trực tiếp chênh lệch kích thước giữa single-stage và multi-stage build.

**Bước 1.** Tạo ứng dụng Go tối giản:

```bash
mkdir -p ~/lab10-multistage && cd ~/lab10-multistage
cat > go.mod << 'EOF'
module lab10multistage

go 1.23
EOF
cat > main.go << 'EOF'
package main

import "fmt"

func main() {
	fmt.Println("Multi-stage build lab")
}
EOF
```

**Bước 2.** Viết Dockerfile single-stage (để so sánh):

```dockerfile
# Dockerfile.single
FROM golang:1.23
WORKDIR /src
COPY . .
RUN go build -o /out/app .
CMD ["/out/app"]
```

**Bước 3.** Viết Dockerfile multi-stage:

```dockerfile
# Dockerfile.multi
FROM golang:1.23 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

FROM alpine:3.20
COPY --from=builder /out/app /app
ENTRYPOINT ["/app"]
```

**Bước 4.** Build cả hai, so sánh kích thước:

```bash
docker build -f Dockerfile.single -t lab10:single .
docker build -f Dockerfile.multi -t lab10:multi .
docker images | grep lab10
```

**Yêu cầu hoàn thành:** ghi lại kích thước cụ thể của cả 2 image (số thật đo được trên máy bạn, không suy đoán), giải thích chênh lệch dựa trên nội dung từng stage.

## Lab 5 (Advanced) — BuildKit cache mount, đo tốc độ build lặp lại

**Mục tiêu:** thực hành cache mount cho dependency, đo tốc độ build lần 2 trở đi.

**Bước 1.** Dùng lại project Node.js ở Lab 2, thêm vài dependency thật:

```bash
cd ~/lab10-cache
cat > package.json << 'EOF'
{
  "name": "lab10-cache",
  "version": "1.0.0",
  "dependencies": { "express": "^4.19.2" }
}
EOF
```

**Bước 2.** Viết Dockerfile dùng cache mount:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine
WORKDIR /app
COPY package.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm install
COPY . .
CMD ["node", "server.js"]
```

**Bước 3.** Build lần đầu, xóa image (không xóa cache mount), build lại, so sánh thời gian:

```bash
time docker build -t lab10-cachemount:1.0 .
docker rmi lab10-cachemount:1.0
time docker build -t lab10-cachemount:1.0 .
```

**Bước 4.** Thử `docker builder prune --all --force` rồi build lại lần nữa, so sánh thời gian với 2 lần trước.

**Yêu cầu hoàn thành:** giải thích bằng dữ liệu đo được: vì sao lần build thứ 2 (sau khi xóa image nhưng chưa prune cache) vẫn nhanh hơn đáng kể so với lần build đầu và lần build sau khi prune.

## Lab 6 (Advanced/Mini Project) — Đóng gói một service thực tế, chuẩn production

**Mục tiêu:** tổng hợp toàn bộ kiến thức module vào một Dockerfile hoàn chỉnh, đạt chuẩn có thể review trong môi trường doanh nghiệp.

**Yêu cầu:**

1. Chọn 1 trong 2 stack: Python (Flask/FastAPI) hoặc Node.js (Express) — viết một API service tối giản có ít nhất 2 endpoint, trong đó có `/health`.
2. Viết Dockerfile **multi-stage** thỏa mãn TẤT CẢ điều kiện sau:
   - `FROM` pin version cụ thể ở cả 2 stage.
   - Stage build tách biệt hoàn toàn khỏi stage final (không copy source code thừa, không copy dev dependency).
   - Dùng `--mount=type=cache` cho bước cài dependency.
   - `USER` non-root ở stage final.
   - `COPY --chown=` để set quyền đúng ngay lúc copy.
   - `HEALTHCHECK` trỏ đúng endpoint `/health`.
   - `LABEL` ghi rõ `maintainer` và `version`.
   - `.dockerignore` loại trừ `.git`, file `.env`, thư mục dependency local.
   - `ENTRYPOINT` + `CMD` kết hợp đúng chuẩn (exec form).
3. Build image, gán tag theo quy ước `<tên-service>:<semver>`.
4. Chạy `docker history --no-trunc <image>` và tự đánh giá: layer nào có thể gộp thêm để giảm số lượng, layer nào đang chiếm dung lượng nhiều nhất.
5. Chạy container, xác nhận healthcheck chuyển `healthy`, xác nhận `docker exec ... whoami` không phải root.
6. Đo kích thước image cuối cùng bằng `docker images`, ghi lại con số thật.

**Sản phẩm nộp (lưu trong [[labs/README|labs/]]):** Dockerfile hoàn chỉnh, `.dockerignore`, source code service, và một ghi chú ngắn (không phải câu hỏi phỏng vấn) mô tả: kích thước image đạt được, các quyết định thiết kế (vì sao chọn base image này, vì sao tách stage như vậy).

Tiếp theo: [[06-Troubleshooting]] để chuẩn bị xử lý các lỗi build thường gặp khi làm lab và trong công việc thực tế.
