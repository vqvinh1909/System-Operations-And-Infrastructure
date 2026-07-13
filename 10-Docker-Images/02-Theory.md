---
title: "02 - Theory"
module: 10
tags: [docker, sysops-infra, module-10, theory, dockerfile]
---

# 02 - Theory: Dockerfile Instructions, Layer, Cache

## 1. Nguyên lý nền: mỗi instruction = một layer

Trước khi đi vào từng instruction, cần nắm nguyên lý cốt lõi chi phối toàn bộ chương này:

> Mỗi instruction trong Dockerfile (trừ vài instruction chỉ set metadata) khi build sẽ tạo ra **một layer** mới, xếp chồng lên layer trước đó, theo dạng **union filesystem** (thường là overlay2 trên Linux — nhắc lại từ Module 09).

Layer là **read-only** và được nhận diện bằng một **content hash**. Nếu instruction và context của nó không đổi, BuildKit tái sử dụng layer đã build trước đó thay vì build lại — đây chính là **cache**. Toàn bộ [[04-Commands]] và [[06-Troubleshooting]] sau này xoay quanh việc tận dụng đúng nguyên lý này.

### Internal Working: điều gì xảy ra khi bạn gõ `docker build`

```
1. Docker CLI đóng gói "build context" (thư mục bạn chỉ định, thường là ".")
   gửi lên BuildKit (qua daemon hoặc buildx), trừ các file khớp .dockerignore.
2. BuildKit đọc Dockerfile, dựng thành DAG (Directed Acyclic Graph) các bước,
   không nhất thiết chạy tuần tự 100% — các bước độc lập có thể chạy song song.
3. Với mỗi bước (instruction), BuildKit tính cache key dựa trên:
   - Instruction đó + tham số của nó
   - Cache key của layer cha (layer ngay trước nó)
   - Với COPY/ADD: checksum nội dung file được copy
4. Nếu cache key trùng với lần build trước -> cache HIT -> tái sử dụng layer cũ,
   bỏ qua thực thi.
5. Nếu cache MISS -> chạy instruction thật (RUN thì exec lệnh trong container tạm),
   snapshot filesystem diff thành layer mới, gán cache key mới.
6. Layer con luôn MISS nếu layer cha đã MISS (hiệu ứng domino) -> đây là lý do
   THỨ TỰ instruction trong Dockerfile cực kỳ quan trọng.
7. Sau khi chạy hết instruction, các layer được xếp chồng + file cấu hình (config)
   -> đóng gói thành 1 image tuân theo OCI Image Spec (manifest + layer + config).
```

Đây chính là lý do một Dockerfile viết ẩu (ví dụ `COPY . .` ngay từ đầu, trước khi cài dependency) khiến MỌI lần sửa 1 dòng code đều làm mất cache của toàn bộ các bước cài dependency phía sau — dù dependency không hề đổi.

## 2. FROM — chọn base image

```dockerfile
FROM node:20-alpine
```

`FROM` luôn là instruction đầu tiên (trừ khi dùng `ARG` trước `FROM` để tham số hóa tag — sẽ nói ở phần ARG). Nó xác định layer nền (base layer) mà mọi layer sau xếp chồng lên.

**Vì sao base image quan trọng bậc nhất:**
- Kích thước base image chiếm phần lớn kích thước image cuối (`ubuntu:22.04` ~ 78MB, `alpine:3.20` ~ 7MB, `distroless` còn nhỏ hơn nữa — không có shell, không có package manager).
- Base image mang theo toàn bộ CVE (lỗ hổng bảo mật) của nó. Base càng ít package, bề mặt tấn công càng nhỏ.
- **Pin version cụ thể** (`node:20.15-alpine`, không dùng `node:latest`) để build có tính **tái lập** (reproducible) — tránh tình huống kinh điển "build hôm qua chạy được, build hôm nay lỗi" vì `latest` đã đổi.

`FROM scratch` là trường hợp đặc biệt: không có base nào cả, dùng khi build image chỉ chứa 1 binary tĩnh (static binary, ví dụ Go compiled với `CGO_ENABLED=0`).

Multi-stage build dùng `FROM ... AS <tên-stage>` để đặt tên cho một stage, phục vụ `COPY --from=` sau này (chi tiết ở mục Multi-stage bên dưới và [[03-Architecture]]).

## 3. RUN — thực thi lệnh, tạo layer

```dockerfile
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

`RUN` thực thi lệnh shell **trong lúc build**, kết quả (file thay đổi trên filesystem) được snapshot thành 1 layer.

**Internal Working:** BuildKit khởi tạo một container tạm từ layer hiện tại, chạy lệnh trong đó, rồi so sánh filesystem trước/sau để tính ra **diff** — đó chính là nội dung layer mới. Container tạm này bị hủy sau khi RUN xong, chỉ có diff được giữ lại.

**Nguyên tắc bắt buộc: gộp RUN liên quan bằng `&&`.** Vì mỗi RUN tạo 1 layer, và **file bị xóa ở RUN sau không làm giảm kích thước layer trước** (layer trước đã đóng băng, immutable). Ví dụ sai:

```dockerfile
# SAI — layer 1 vẫn giữ nguyên cache apt dù layer 2 xóa nó
RUN apt-get update && apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

```dockerfile
# ĐÚNG — cache apt được dọn trong CÙNG layer, không bị giữ lại
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

Có 2 dạng cú pháp: **shell form** (`RUN lệnh`, chạy qua `/bin/sh -c`) và **exec form** (`RUN ["lệnh", "arg1"]`, chạy trực tiếp không qua shell, không xử lý biến môi trường/wildcard).

## 4. COPY vs ADD — khác biệt bắt buộc phải nhớ

```dockerfile
COPY package.json /app/
ADD app.tar.gz /app/
```

| | COPY | ADD |
|---|---|---|
| Copy file/thư mục local | Có | Có |
| Tự giải nén archive (tar, gz...) | Không | **Có** (tự động extract) |
| Tải file từ URL | Không | Có (nhưng **không khuyến khích**) |
| Độ minh bạch | Cao — làm đúng những gì thấy | Thấp — hành vi "ẩn" (auto-extract, auto-download) |

**Khuyến nghị thực chiến (và cũng là khuyến nghị chính thức của Docker): luôn ưu tiên `COPY`, chỉ dùng `ADD` khi thực sự cần tính năng tự giải nén archive local.** Lý do: `ADD` với URL không hỗ trợ xóa file sau khi tải (phải RUN thêm để dọn, tốn layer), và hành vi tự giải nén dễ gây bất ngờ cho người đọc Dockerfile sau này — vi phạm nguyên tắc "Dockerfile phải tường minh, dễ audit".

## 5. WORKDIR — thư mục làm việc

```dockerfile
WORKDIR /app
```

Set thư mục làm việc cho mọi instruction sau đó (`RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD`). Tương đương `cd /app` nhưng **persist qua các layer** (khác với `RUN cd /app` — chỉ có hiệu lực trong chính RUN đó vì mỗi RUN chạy trong shell process riêng).

Best practice: dùng `WORKDIR` thay vì `RUN mkdir -p /app && cd /app` — WORKDIR tự tạo thư mục nếu chưa tồn tại, và tường minh hơn.

## 6. ENV vs ARG — khác biệt quan trọng nhất hay bị hỏi phỏng vấn

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

ARG BUILD_ENV=production
ENV NODE_ENV=${BUILD_ENV}
```

| | ARG | ENV |
|---|---|---|
| Tồn tại lúc nào | **Chỉ lúc build** | Lúc build **và** lúc container chạy |
| Truy cập từ container đang chạy | Không (`docker exec ... env` không thấy) | Có |
| Cách set | `--build-arg KEY=value` khi `docker build` | Cố định trong Dockerfile hoặc kế thừa từ ARG |
| Phạm vi | Từng stage riêng (multi-stage phải khai báo lại nếu dùng qua nhiều stage) | Toàn bộ layer từ điểm khai báo trở đi |

**Internal Working quan trọng:** `ARG` khai báo **trước `FROM`** có phạm vi đặc biệt — chỉ dùng được để tham số hóa chính dòng `FROM`, và **không tự động mang vào bên trong stage**; muốn dùng tiếp bên trong stage phải khai báo lại `ARG` sau `FROM`.

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine
ARG NODE_VERSION
RUN echo "Building with Node ${NODE_VERSION}"
```

**Cảnh báo bảo mật:** Giá trị `ARG` **vẫn lưu lại trong build history/metadata của image** (xem được bằng `docker history`), dù không tồn tại lúc runtime. **Tuyệt đối không dùng ARG để truyền secret** (password, API key) — dùng BuildKit secret mount (`--mount=type=secret`, xem [[04-Commands]]) thay thế.

## 7. ENTRYPOINT và CMD — khác biệt và cách kết hợp

Đây là 2 instruction dễ nhầm nhất với người mới, vì cả hai đều "định nghĩa lệnh chạy khi container start", nhưng vai trò khác nhau:

- **ENTRYPOINT**: lệnh **cố định**, luôn chạy. Định nghĩa "container này là gì" (ví dụ luôn là một binary cụ thể).
- **CMD**: **tham số mặc định**, có thể bị **ghi đè** bởi argument truyền vào lúc `docker run image <args>`.

**Cách kết hợp phổ biến nhất trong thực chiến:**

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

Khi chạy `docker run myimage`, lệnh thực thi là `nginx -g daemon off;`. Khi chạy `docker run myimage -v`, CMD bị ghi đè, lệnh thực thi thành `nginx -v` — ENTRYPOINT không đổi, chỉ CMD (tham số) bị thay.

**Bảng hành vi 4 tổ hợp (exec form là bắt buộc để có hành vi này đúng — shell form không kết hợp được):**

| ENTRYPOINT | CMD | Khi `docker run image` | Khi `docker run image arg1` |
|---|---|---|---|
| Không có | `["node", "server.js"]` | `node server.js` | `arg1` (ghi đè toàn bộ CMD) |
| `["node", "server.js"]` | Không có | `node server.js` | `node server.js arg1` (arg1 nối thêm) |
| `["nginx"]` | `["-g", "daemon off;"]` | `nginx -g daemon off;` | `nginx arg1` (CMD bị ghi đè) |
| shell form bất kỳ | — | Chạy qua `/bin/sh -c`, **không nhận argument override kiểu exec form** | — |

**Shell form vs exec form:**

```dockerfile
CMD node server.js        # shell form: chạy qua /bin/sh -c "node server.js"
CMD ["node", "server.js"] # exec form: chạy trực tiếp, PID 1 là node
```

**Vì sao exec form được khuyến nghị:** với shell form, tiến trình `node` chạy như **con của `/bin/sh`**, không phải PID 1. Hệ quả: khi Docker gửi signal `SIGTERM` để dừng container êm (graceful shutdown), signal đó tới `/bin/sh` chứ không tới `node`, khiến ứng dụng không kịp cleanup, Docker phải đợi hết `--stop-timeout` (mặc định 10 giây) rồi `SIGKILL` cưỡng bức. Đây là nguyên nhân phổ biến của tình huống "container mất vài giây/chục giây mới dừng" trong thực tế vận hành.

## 8. EXPOSE — chỉ là tài liệu, không publish port thật

```dockerfile
EXPOSE 8080
```

**Hiểu nhầm phổ biến nhất về EXPOSE:** nhiều người tưởng `EXPOSE` sẽ "mở port ra ngoài". **Sai.** `EXPOSE` chỉ là **metadata/tài liệu**, khai báo "ứng dụng bên trong container lắng nghe ở port này" — không tự động publish port ra host. Port chỉ thực sự được publish khi chạy `docker run -p <host-port>:<container-port>`.

Tác dụng thực tế của `EXPOSE`:
- Tài liệu hóa cho người đọc Dockerfile / dùng image.
- Nếu chạy `docker run -P` (chữ P hoa), Docker sẽ tự publish **các port đã EXPOSE** ra port ngẫu nhiên trên host.
- Một số tool orchestration (Compose, một số IDE) đọc EXPOSE để gợi ý cấu hình network.

## 9. USER — không chạy container bằng root

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**Đây là một trong những best practice bảo mật quan trọng nhất, và cũng là lỗi phổ biến nhất bị security team "bắt lỗi" khi audit image.**

Nếu Dockerfile không có `USER`, container mặc định chạy bằng **root** (UID 0) bên trong container. Vấn đề: dù namespace cô lập container khỏi host, nhưng nếu kẻ tấn công khai thác được lỗ hổng để **thoát container (container breakout)**, họ sẽ có quyền root ngay trên host — hậu quả nghiêm trọng hơn nhiều so với chỉ có quyền user thường.

**Nguyên tắc least privilege** (đã học ở chuỗi Security Hardening Linux) áp dụng y hệt cho container: luôn tạo user riêng, không có quyền sudo, chỉ đủ quyền chạy ứng dụng.

```dockerfile
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["node", "server.js"]
```

Lưu ý: `COPY --chown=` giúp set quyền sở hữu file ngay lúc copy, tránh phải thêm 1 layer `RUN chown` riêng (vừa tốn layer, vừa tốn dung lượng vì `chown` trên nhiều file tạo diff lớn).

## 10. LABEL — metadata có cấu trúc

```dockerfile
LABEL maintainer="devops@company.com" \
      version="1.4.2" \
      description="Internal billing API service"
```

`LABEL` gắn metadata dạng key-value vào image, xem được bằng `docker inspect`. Dùng cho: theo dõi nguồn gốc build (Git commit, CI build number), phân loại image trong registry nội bộ, hoặc để tool tự động hóa (ví dụ script dọn dẹp image cũ) lọc theo label.

## 11. VOLUME — khai báo mount point

```dockerfile
VOLUME /var/lib/mysql
```

Khai báo một thư mục trong container là **mount point** cho volume — Docker sẽ tự tạo anonymous volume gắn vào đó nếu người chạy container không tự chỉ định volume/bind mount. Mục đích: đảm bảo dữ liệu ở thư mục này **không bị mất** khi container bị xóa, và **không bị ghi vào writable layer** (vốn có hiệu năng I/O kém hơn volume, đặc biệt trên storage driver overlay2).

**Lưu ý thực chiến:** một khi đã `VOLUME` trong Dockerfile, **không có `RUN` nào sau đó có thể sửa nội dung thư mục ấy có hiệu lực trong image** — vì tại thời điểm container thật sự chạy, Docker sẽ mount đè volume lên, nội dung ghi lúc build bị che khuất. Vì vậy `VOLUME` nên đặt **sau cùng**, sau khi mọi RUN cần ghi vào thư mục đó đã hoàn tất.

## 12. HEALTHCHECK — Docker tự kiểm tra sức khỏe container

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

Định nghĩa lệnh Docker daemon định kỳ chạy **bên trong container** để xác định container có "healthy" hay không. Trạng thái hiển thị qua `docker ps` (cột STATUS: `healthy`/`unhealthy`/`starting`).

**Vì sao quan trọng trong doanh nghiệp:** orchestrator (Swarm, Kubernetes qua readiness/liveness probe tương tự) dựa vào healthcheck để quyết định có nên route traffic vào container này không, hoặc có nên tự động restart nó không. Container "chạy" (process còn sống, `docker ps` thấy `Up`) không đồng nghĩa với "sẵn sàng phục vụ" — ví dụ ứng dụng Java còn đang warm up JVM, hoặc kết nối DB bị treo.

Tham số quan trọng:
- `--interval`: chu kỳ kiểm tra.
- `--timeout`: thời gian chờ tối đa 1 lần check.
- `--start-period`: thời gian "ân hạn" lúc mới start, healthcheck fail trong giai đoạn này không tính là unhealthy (dành cho app khởi động chậm).
- `--retries`: số lần fail liên tiếp trước khi đánh dấu unhealthy.

## 13. SHELL — đổi shell mặc định cho instruction dạng shell form

```dockerfile
SHELL ["/bin/bash", "-c"]
```

Mặc định, shell form của `RUN`/`CMD`/`ENTRYPOINT` chạy qua `/bin/sh -c` trên Linux. `SHELL` cho phép đổi sang shell khác (ví dụ `bash` để dùng cú pháp bash-specific như `[[ ]]`, `pipefail`). Ít dùng trong Dockerfile Linux thông thường, phổ biến hơn trên Windows container (đổi sang `powershell`).

## 14. STOPSIGNAL — signal dùng để dừng container

```dockerfile
STOPSIGNAL SIGTERM
```

Định nghĩa signal mà Docker gửi tới PID 1 trong container khi chạy `docker stop`. Mặc định là `SIGTERM`. Một số ứng dụng (ví dụ Nginx dùng `SIGQUIT` để graceful shutdown thay vì `SIGTERM`) cần khai báo lại để `docker stop` dừng đúng cách thay vì đợi timeout rồi bị `SIGKILL`.

## 15. ONBUILD — instruction "hoãn" cho image con

```dockerfile
ONBUILD COPY . /app/src
ONBUILD RUN make /app/build
```

`ONBUILD` đăng ký một instruction sẽ **không chạy ngay** trong lần build image này, mà chỉ chạy khi có một **image khác dùng image này làm `FROM`**. Dùng để tạo "base image có sẵn kịch bản build", ví dụ image ngôn ngữ lập trình chuẩn hóa quy trình build cho mọi team.

**Lưu ý thực chiến:** `ONBUILD` khá hiếm gặp trong Dockerfile hiện đại — phần lớn tổ chức chuyển sang dùng multi-stage build hoặc CI pipeline template thay vì ONBUILD, vì ONBUILD làm hành vi build "ẩn" (không thấy ngay trong Dockerfile của image con), khó audit và dễ gây bất ngờ.

## 16. Layer Cache — cơ chế và nguyên tắc sắp xếp Dockerfile

Đã nói ở mục 1, nhắc lại có hệ thống:

**Nguyên tắc vàng: sắp xếp instruction từ ít thay đổi nhất → thay đổi thường xuyên nhất.**

```dockerfile
# Thứ tự ĐÚNG cho app Node.js
FROM node:20-alpine        # 1. Gần như không đổi
WORKDIR /app                # 2. Không đổi
COPY package*.json ./       # 3. Chỉ đổi khi thêm/bớt dependency
RUN npm ci --omit=dev       # 4. Cache HIT nếu package*.json không đổi
COPY . .                    # 5. Đổi MỖI lần sửa code -> đặt SAU CÙNG
CMD ["node", "server.js"]
```

Nếu đảo ngược — `COPY . .` trước `RUN npm ci` — thì **mỗi lần sửa 1 dòng code** (dù không đụng tới dependency) sẽ làm cache miss ngay từ bước COPY, kéo theo `RUN npm ci` (bước tốn thời gian nhất, tải cả trăm package) phải chạy lại từ đầu mỗi lần build. Đây là nguyên nhân phổ biến nhất khiến CI/CD build chậm bất thường trong thực tế — sẽ mổ xẻ case cụ thể ở [[06-Troubleshooting]].

### Cache invalidation — khi nào cache bị "vô hiệu"

- `RUN`: cache dựa trên **chuỗi lệnh** (text) — chỉ cần đổi 1 ký tự là miss, dù kết quả thực thi giống hệt.
- `COPY`/`ADD`: cache dựa trên **checksum nội dung file/thư mục** được copy — đổi nội dung, đổi permission, hoặc đổi mtime (tùy driver) đều có thể gây miss.
- Một khi 1 layer miss, **toàn bộ layer phía sau nó đều miss theo** (hiệu ứng domino, vì cache key của layer con phụ thuộc cache key của layer cha).
- `--no-cache` khi build sẽ bỏ qua toàn bộ cache, build lại từ đầu — dùng khi nghi ngờ cache "sai" (ví dụ base image tag `latest` đã đổi nhưng Docker vẫn tưởng còn hợp lệ).

## 17. Multi-stage Build — tại sao và cách hoạt động

**Vấn đề cần giải quyết:** một image build ra thường cần nhiều công cụ chỉ dùng **lúc build** (compiler, dev dependency, source code, build cache) nhưng **không cần lúc chạy** (runtime chỉ cần binary/artifact cuối cùng). Nếu build tất cả trong 1 stage, mọi thứ (kể cả compiler nặng hàng trăm MB) đều nằm lại trong image production.

**Giải pháp:** multi-stage build cho phép Dockerfile có **nhiều `FROM`**, mỗi `FROM` là một stage độc lập, và stage sau có thể `COPY --from=<stage>` để lấy **chỉ những file cần thiết** từ stage trước, bỏ lại toàn bộ phần còn lại.

```dockerfile
# Stage 1: builder — chứa đầy đủ toolchain Go
FROM golang:1.23 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/server .

# Stage 2: final — chỉ chứa binary + base tối giản
FROM alpine:3.20
RUN adduser -D appuser
COPY --from=builder /out/server /app/server
USER appuser
ENTRYPOINT ["/app/server"]
```

Kết quả: image cuối chỉ chứa Alpine base (~7MB) + 1 binary tĩnh, **không hề có Go compiler, source code, hay module cache** — dù toàn bộ quá trình build vẫn cần chúng ở stage 1. Xem sơ đồ trực quan ở [[03-Architecture]].

**Lợi ích cụ thể trong doanh nghiệp:**
- Giảm kích thước image → kéo image nhanh hơn trên mọi node, tiết kiệm băng thông registry nội bộ, giảm chi phí lưu trữ.
- Giảm bề mặt tấn công → image production không có compiler, không có source code (tránh leak mã nguồn nếu image bị lộ), không có dev dependency có thể chứa CVE.
- Tách rõ trách nhiệm: stage build có thể dùng image nặng, nhiều công cụ debug; stage final tối giản, chuẩn hóa cho production.

Có thể có nhiều hơn 2 stage (ví dụ stage riêng cho chạy test, stage riêng cho lint), và `COPY --from=` có thể lấy từ **image bên ngoài** (không chỉ stage trong cùng Dockerfile), ví dụ `COPY --from=nginx:1.27 /etc/nginx/nginx.conf /tmp/`.

## 18. BuildKit — kiến trúc build hiện đại

BuildKit là builder mặc định cho `docker build` trên Linux từ Docker Engine 23.0 trở đi (trước đó cần bật thủ công qua biến môi trường `DOCKER_BUILDKIT=1`). So với legacy builder cũ:

| Đặc điểm | Legacy builder | BuildKit |
|---|---|---|
| Thực thi | Tuần tự từng instruction | Dựng DAG, chạy song song các nhánh độc lập |
| Cache | Chỉ theo layer tuần tự | Cache key theo content hash, linh hoạt hơn, hỗ trợ cache mount |
| Secret trong build | Không có cơ chế an toàn (dễ leak vào layer) | `--mount=type=secret` — secret không lưu vào layer |
| Build đa nền tảng | Không hỗ trợ trực tiếp | `docker buildx` — build multi-arch (amd64/arm64) trong 1 lần |
| Output | Chỉ ra image | Có thể xuất ra local dir, tar, hoặc build không tạo image (`--output=type=local`) |

**Cache mount (`--mount=type=cache`)** là tính năng thực chiến quan trọng: cho phép một `RUN` gắn một thư mục cache **tồn tại xuyên suốt nhiều lần build** (không nằm trong layer, không làm tăng kích thước image), ví dụ cache `apt`, `pip`, `npm`, Go module cache, Maven `.m2`. Cú pháp và ví dụ đầy đủ ở [[04-Commands]].

**Secret mount (`--mount=type=secret`)** cho phép truyền secret (API token, private key để `git clone` repo private) vào **trong RUN** mà **không lưu lại** trong bất kỳ layer nào của image cuối — khác hẳn với việc dùng `ARG`/`ENV` (vẫn còn dấu vết trong `docker history`).

**buildx** là CLI plugin mở rộng cho `docker build`, cho phép chọn builder (BuildKit instance riêng, có thể chạy remote hoặc trong container), build multi-platform, và xuất output linh hoạt.

## 19. Best Practices tổng hợp

- **Base image nhỏ, pin version**: ưu tiên `alpine`, `slim`, hoặc `distroless` (Google distroless — không có shell, không package manager, giảm tối đa bề mặt tấn công); luôn pin tag cụ thể, tránh `latest`.
- **Gộp RUN liên quan, dọn cache trong cùng layer** (mục 3).
- **`.dockerignore`**: loại trừ file không cần thiết khỏi build context (`.git`, `node_modules`, file log, secret local) — vừa giảm thời gian gửi context lên daemon, vừa tránh leak file nhạy cảm vào image nếu lỡ `COPY .`.
- **Không chạy root** — luôn có `USER` (mục 9).
- **Thứ tự COPY hợp lý** để tận dụng cache (mục 16).
- **Multi-stage build** cho mọi ngôn ngữ compiled hoặc có bước build riêng (mục 17).
- **Không đưa secret vào ARG/ENV/COPY** — dùng BuildKit secret mount.
- **HEALTHCHECK** cho service chạy dài hạn, để orchestrator biết trạng thái thật.
- **LABEL** đầy đủ metadata để truy vết nguồn gốc image trong registry nội bộ.

Tiếp theo: [[03-Architecture]] để xem sơ đồ trực quan hóa layer stacking và multi-stage build.
