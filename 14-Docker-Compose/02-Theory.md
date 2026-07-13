---
title: "02 - Theory"
module: 14
tags: [docker, sysops-infra, module-14, theory, internal-working]
---

# 02 - Lý thuyết Docker Compose

## 1. Compose Specification là gì

Trước đây (Compose CLI v1 và các bản Compose v2 đời đầu), có sự phân mảnh giữa "version 2.x" và "version 3.x" của file compose, mỗi version hỗ trợ tập tính năng khác nhau, gây nhầm lẫn rất nhiều — ví dụ `deploy` chỉ dùng được ở version 3.x với Swarm, `depends_on` dạng có điều kiện lại chỉ có ở version 2.1+.

Từ khi cộng đồng hợp nhất thành **Compose Specification** (một đặc tả mở, độc lập với việc chạy trên Docker Engine hay engine khác), sự phân mảnh version bị xóa bỏ. Compose CLI hiện tại (nhánh 5.x, ví dụ v5.3.1) luôn dùng schema **mới nhất** để validate file, bất kể khóa `version:` ghi gì bên trong — vì vậy:

- Khóa `version:` ở đầu file **không còn tác dụng gì**, Compose bỏ qua (ignore) hoàn toàn, chỉ để lại cảnh báo "obsolete" nếu vẫn còn.
- Best practice 2026: **bỏ hẳn khóa `version:`** ra khỏi file.
- Tên file khuyến nghị: **`compose.yaml`** (không phải `docker-compose.yml` — tên cũ vẫn được hỗ trợ để tương thích ngược, nhưng tài liệu hiện tại dùng `compose.yaml` làm chuẩn).

## 2. Cấu trúc top-level của compose.yaml

```yaml
services:    # bắt buộc — định nghĩa từng container/service của ứng dụng
networks:    # optional — khai báo network tùy chỉnh
volumes:     # optional — khai báo named volume
secrets:     # optional — khai báo secret (file-based hoặc từ biến môi trường)
configs:     # optional — khai báo file cấu hình chia sẻ cho service (tương tự secrets nhưng không cần bảo mật cao)
profiles:    # optional — (ở cấp service) gắn service vào 1 hay nhiều "hồ sơ" bật/tắt được
include:     # optional — gộp (include) compose file khác vào file hiện tại
name:        # optional — đặt tên project tường minh, thay vì suy ra từ tên thư mục
```

Chỉ `services` là bắt buộc — một file compose.yaml hợp lệ tối thiểu chỉ cần:

```yaml
services:
  web:
    image: nginx:1.27
```

## 3. Services — trái tim của compose.yaml

### 3.1. `build` vs `image`

Hai cách một service có được image để chạy:

```yaml
services:
  # Cách 1: dùng image có sẵn (kéo từ registry)
  db:
    image: postgres:16

  # Cách 2: build từ Dockerfile trong thư mục hiện tại
  backend:
    build:
      context: ./backend        # thư mục chứa Dockerfile và build context
      dockerfile: Dockerfile    # tên file Dockerfile (mặc định là "Dockerfile")
    image: myapp-backend:1.0    # đặt tên/tag cho image build ra (khuyến khích, để dễ trace log)
```

**Nguyên tắc chọn:** Trong môi trường production/staging thực tế, image thường được build sẵn ở pipeline CI, đẩy lên registry nội bộ (Harbor, ECR, GitLab Registry — sẽ học ở Module 16-Image-Registry), rồi Compose ở server chỉ `pull` về bằng `image:`. Dùng `build:` trực tiếp trong compose.yaml chủ yếu phù hợp cho **môi trường dev** — sửa code, `docker compose up --build`, test ngay tại chỗ, không cần qua registry.

### 3.2. `depends_on` — và cái bẫy phổ biến nhất

Cú pháp ngắn (short syntax) chỉ đảm bảo **thứ tự container start**, không đảm bảo dependency đã **sẵn sàng phục vụ**:

```yaml
services:
  backend:
    depends_on:
      - db   # chỉ đảm bảo container db được start TRƯỚC backend, KHÔNG đảm bảo Postgres đã accept connection
```

Đây chính là nguyên nhân phổ biến nhất gây lỗi "connection refused" khi `docker compose up` lần đầu: container `db` được Docker start, nhưng tiến trình PostgreSQL bên trong còn đang khởi tạo (init database, load WAL...) mất vài giây, trong khi container `backend` đã start ngay sau đó và cố kết nối — connect fail, backend crash, tùy `restart` policy có thể crash loop.

Cách giải quyết đúng: kết hợp `depends_on` dạng **long syntax** với `condition: service_healthy`, để backend chỉ start khi healthcheck của db báo "healthy":

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s

  backend:
    build: ./backend
    depends_on:
      db:
        condition: service_healthy   # chờ healthcheck của db pass mới start backend
```

Các giá trị `condition` khác:
- `service_started` — hành vi mặc định (tương đương short syntax), chỉ chờ container start.
- `service_healthy` — chờ container đạt trạng thái `healthy` theo `healthcheck`.
- `service_completed_successfully` — chờ container (thường là job một lần, ví dụ chạy migration) exit với code 0.

### 3.3. `restart` policy trong Compose

```yaml
services:
  backend:
    restart: unless-stopped
```

Các giá trị: `no` (mặc định), `always`, `on-failure`, `unless-stopped`. Ý nghĩa giống hệt flag `--restart` của `docker run` đã học ở Module 09 — Compose chỉ là cách khai báo nó trong YAML thay vì gõ flag.

## 4. Networks — Compose tự tạo network như thế nào

### 4.1. Default network

Nếu compose.yaml **không khai báo** `networks:` gì cả, Compose tự động:

1. Tạo một network kiểu `bridge` đặt tên `<project_name>_default`.
2. Gắn (attach) **mọi service trong file** vào network này.
3. Trong network đó, Compose tự cấu hình DNS nội bộ — mỗi service gọi tới service khác **bằng chính tên service** (không cần biết IP). Ví dụ backend kết nối `db` chỉ cần dùng hostname `db`, Compose tự resolve ra đúng IP container.

Đây là lý do trong ví dụ ở mục 3.2, backend chỉ cần cấu hình `DB_HOST=db` thay vì phải biết IP container — điều không thể làm được nếu dùng `docker run` rời rạc mà không tự tạo network chung.

### 4.2. Custom network

Khai báo tường minh khi cần nhiều network tách biệt (ví dụ tách network "frontend" — chỉ nginx và backend thấy nhau, và network "backend" — chỉ backend và db thấy nhau, để db không lộ ra ngoài):

```yaml
services:
  nginx:
    networks: [frontend]
  backend:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
```

Mô hình này là thực hành bảo mật thật sự dùng trong doanh nghiệp: container `db` không nằm trong network `frontend`, nên dù `nginx` bị compromise, kẻ tấn công cũng không thể route thẳng tới `db`.

### 4.3. External network

Dùng khi cần nhiều project Compose khác nhau (chạy độc lập, up/down riêng) chia sẻ chung một network đã tồn tại từ trước (ví dụ network do một Compose project khác tạo, hoặc tạo thủ công bằng `docker network create`):

```yaml
networks:
  shared_net:
    external: true    # Compose KHÔNG tự tạo, chỉ dùng network đã tồn tại; lỗi nếu network chưa có
```

## 5. Volumes trong Compose

### 5.1. Named volume

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data   # tên "db_data" tham chiếu tới named volume khai báo bên dưới

volumes:
  db_data:    # Compose tự tạo volume "<project_name>_db_data" nếu chưa tồn tại
```

Named volume được Docker Engine quản lý (mặc định lưu ở `/var/lib/docker/volumes/` trên Linux), tồn tại độc lập với vòng đời container — `docker compose down` (không kèm `-v`) sẽ xóa container nhưng **giữ nguyên** volume, dữ liệu không mất.

### 5.2. Bind mount

```yaml
services:
  nginx:
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro   # bind mount file cấu hình từ host, chỉ đọc (ro)
      - ./html:/usr/share/nginx/html
```

Bind mount trỏ trực tiếp tới đường dẫn trên host — dùng khi cần **sửa file trên host và thấy hiệu lực ngay trong container** (ví dụ mount code khi dev, hoặc mount file config quản lý bằng Git). Khác với named volume, bind mount không được Compose "sở hữu" — không xuất hiện trong `docker volume ls`, không bị xóa bởi `docker compose down -v`.

## 6. Profiles — bật/tắt nhóm service theo tình huống

Một compose.yaml thực tế trong doanh nghiệp thường có thêm các service phụ trợ không cần chạy mọi lúc: công cụ debug, admin UI quản trị database (Adminer, pgAdmin), service seed dữ liệu test. Gắn `profiles` để các service này **mặc định không chạy**, chỉ chạy khi được yêu cầu tường minh:

```yaml
services:
  backend:
    image: myapp-backend:1.0   # không có profiles -> luôn chạy

  adminer:
    image: adminer
    profiles: ["debug"]        # chỉ chạy khi profile "debug" được kích hoạt
```

```bash
docker compose up -d                    # chỉ chạy backend (adminer bị bỏ qua)
docker compose --profile debug up -d    # chạy cả backend và adminer
```

Cũng có thể kích hoạt qua biến môi trường `COMPOSE_PROFILES=debug` thay vì gõ flag mỗi lần.

## 7. Environment — biến môi trường và thứ tự ưu tiên

### 7.1. Các cách khai báo biến môi trường cho container

```yaml
services:
  backend:
    environment:
      - DB_HOST=db
      - DB_PASSWORD=${DB_PASSWORD}   # lấy giá trị từ shell hoặc file .env để "nội suy" (interpolation) vào YAML
    env_file:
      - .env.backend                  # nạp toàn bộ biến trong file này vào container
```

### 7.2. File `.env` — vị trí quan trọng nhất cần nhớ

Compose tự động nạp file `.env` nằm **cùng thư mục với compose.yaml** (chính xác hơn: thư mục project — xác định bởi `--project-directory`, hoặc thư mục chứa file `-f` đầu tiên, hoặc thư mục hiện tại) để phục vụ **nội suy biến** (`${VAR}`) ngay trong compose.yaml. Đây khác với `env_file:` (mục 7.1) — `.env` mặc định phục vụ việc thay thế biến trong chính file YAML, còn `env_file:` là nạp biến thẳng vào container.

Lỗi rất hay gặp: đặt file `.env` sai vị trí (ví dụ trong thư mục con `config/.env`) rồi thắc mắc vì sao biến không được nội suy — Compose không tự tìm file `.env` ở nơi khác ngoài thư mục project trừ khi chỉ định tường minh bằng `--env-file`.

### 7.3. Thứ tự ưu tiên biến môi trường (từ cao xuống thấp)

1. Biến truyền trực tiếp qua CLI, ví dụ `docker compose run -e VAR=x`.
2. Giá trị trong khóa `environment` hoặc `env_file` **được nội suy** từ shell hoặc file `.env` (`${VAR}`).
3. Giá trị gán cứng ngay trong khóa `environment` của compose.yaml.
4. Giá trị nạp từ file chỉ định trong `env_file`.
5. Giá trị `ENV` khai báo sẵn trong Dockerfile của image.

Ghi nhớ nguyên tắc: **càng "gần" lúc container thực sự chạy (CLI, shell) thì độ ưu tiên càng cao**, càng "xa" (đã đóng cứng từ lúc build image) thì độ ưu tiên càng thấp. Điều này rất giống nguyên tắc `extra-vars` (`-e`) luôn thắng `group_vars`/`host_vars` trong Ansible.

## 8. Secrets — không hardcode password vào compose.yaml

Hardcode password thẳng vào `environment:` trong compose.yaml là một trong những lỗi bảo mật phổ biến nhất — file này thường bị commit vào Git, và `docker inspect` cũng có thể đọc ngược lại biến môi trường của container đang chạy.

Cách làm đúng — **file-based secrets**:

```yaml
services:
  db:
    image: postgres:16
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password   # ứng dụng đọc password từ file, không đọc từ biến môi trường

secrets:
  db_password:
    file: ./secrets/db_password.txt   # nội dung file này chính là giá trị secret
```

Compose sẽ mount nội dung file vào đường dẫn cố định **`/run/secrets/<tên_secret>`** bên trong container (ở đây là `/run/secrets/db_password`), với quyền đọc giới hạn. File `./secrets/db_password.txt` **không commit vào Git** (thêm vào `.gitignore`), chỉ tồn tại trên server hoặc được inject qua pipeline CI/CD lúc deploy.

Lưu ý: không phải ứng dụng nào cũng hỗ trợ đọc secret qua file (`_FILE` suffix là quy ước của một số image chính thức như postgres, mysql). Nếu ứng dụng tự viết, cần code đọc trực tiếp từ `/run/secrets/<name>`.

## 9. Healthcheck — cú pháp chi tiết

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]  # lệnh kiểm tra; CMD-SHELL chạy qua shell, CMD chạy trực tiếp không qua shell
      interval: 5s        # khoảng cách giữa 2 lần kiểm tra
      timeout: 3s          # thời gian tối đa chờ lệnh test trả kết quả, quá thì tính là fail
      retries: 5            # số lần fail liên tiếp cho phép trước khi coi là "unhealthy"
      start_period: 10s      # thời gian "ân hạn" lúc mới start, fail trong giai đoạn này KHÔNG bị tính vào retries
```

Vòng đời trạng thái healthcheck: `starting` -> (test chạy mỗi `interval`) -> nếu pass: `healthy`; nếu fail đủ `retries` lần liên tiếp (sau khi đã qua `start_period`): `unhealthy`.

`start_period` đặc biệt quan trọng với database — Postgres/MySQL có thể mất vài giây đến vài chục giây để khởi tạo lần đầu (init database, chạy migration), nếu không có `start_period` đủ dài, healthcheck có thể báo `unhealthy` oan trong lúc DB vẫn đang khởi tạo bình thường, khiến `depends_on.condition: service_healthy` không bao giờ được thỏa (backend chờ mãi không start).

## 10. Internal Working — Compose vận hành thật sự ra sao

### 10.1. Compose không phải "chạy container" — nó gọi Docker Engine API

Bản chất, `docker compose` (Go binary, cài như CLI plugin) đọc file YAML, parse thành cấu trúc dữ liệu nội bộ, rồi tuần tự gọi các API của Docker Engine (cùng API mà `docker network create`, `docker volume create`, `docker run` gọi phía sau) để tạo network, volume, rồi tạo và start container. Compose không có "daemon" riêng chạy nền — nó là công cụ chạy một lần cho mỗi lệnh (`up`, `down`, `ps`...), không giống Docker Engine daemon (`dockerd`) chạy thường trực.

### 10.2. Project name — chìa khóa đặt tên mọi resource

Mọi resource Compose tạo ra (container, network, volume) đều được tiền tố bằng **project name**, xác định theo thứ tự ưu tiên:

1. Flag `-p <name>` hoặc `--project-name` khi chạy lệnh.
2. Biến môi trường `COMPOSE_PROJECT_NAME`.
3. Khóa `name:` khai báo tường minh trong compose.yaml.
4. **Mặc định: tên thư mục chứa compose.yaml** (viết thường, loại ký tự đặc biệt).

Ví dụ compose.yaml nằm trong thư mục `myshop/`, project name suy ra là `myshop`. Compose đặt tên resource theo quy tắc:

- **Container:** `<project>-<service>-<số_thứ_tự>` (ví dụ `myshop-db-1`) — bản Compose hiện tại dùng dấu gạch ngang `-`; các bản Compose v1 cũ hơn dùng dấu gạch dưới `_` (`myshop_db_1`), đây là lý do khi thấy tài liệu cũ có thể thấy format tên khác.
- **Network mặc định:** `<project>_default` (ví dụ `myshop_default`).
- **Volume:** `<project>_<tên_volume_khai_báo>` (ví dụ khai báo `db_data` -> volume thật tên `myshop_db_data`).

Đây chính là lý do 2 project Compose khác nhau (2 thư mục khác nhau, mỗi thư mục 1 compose.yaml độc lập) **không đụng nhau dù dùng cùng tên service** — vì tên thật sự luôn có tiền tố project name riêng biệt.

### 10.3. `docker compose up` làm gì theo đúng thứ tự

1. Đọc và merge các file compose (nếu có nhiều file `-f`), nạp `.env`, thực hiện nội suy biến (`${VAR}`).
2. Validate cấu trúc theo Compose Specification (schema mới nhất, bỏ qua khóa `version:` nếu có).
3. Tính toán **dependency graph** giữa các service (dựa trên `depends_on`, `links`, tham chiếu network/volume).
4. Tạo network chưa tồn tại (theo thứ tự không phụ thuộc gì).
5. Tạo volume chưa tồn tại.
6. Với mỗi service, theo đúng thứ tự dependency graph: build image (nếu có `build:` và cần build lại) hoặc pull image (nếu chưa có local), tạo container, start container.
7. Nếu service có `depends_on.condition: service_healthy`, Compose **chờ** container đó đạt trạng thái `healthy` (poll theo `interval` của healthcheck) trước khi tạo/start container phụ thuộc.

### 10.4. `docker compose up` chạy lại — vì sao đôi khi "không có gì xảy ra"

Compose so sánh **cấu hình mong muốn** (từ compose.yaml hiện tại) với **trạng thái thực tế** (container đang chạy, được biết qua label Compose gắn vào lúc tạo — mỗi container Compose tạo ra đều có label `com.docker.compose.config-hash` lưu hash cấu hình lúc tạo nó). Nếu hash cấu hình không đổi, Compose **không tạo lại container** — chỉ đơn giản để nguyên hoặc start nếu đang dừng. Đây là lý do sửa `compose.yaml` xong chạy lại `docker compose up -d` nhiều khi container cũ vẫn đứng yên như cũ nếu Compose nhận nhầm là không đổi (thường gặp khi chỉ sửa file mount bind — nội dung file đổi nhưng đường dẫn mount trong YAML không đổi, nên hash cấu hình không đổi, container không được recreate). Cách xử lý triệt để nằm ở [[06-Troubleshooting]].

## Tiếp theo

Sang [[03-Architecture]] để xem sơ đồ trực quan hóa toàn bộ mối quan hệ project - service - network - volume.
