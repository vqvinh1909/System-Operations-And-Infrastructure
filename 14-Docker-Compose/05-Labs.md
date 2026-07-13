---
title: "05 - Labs"
module: 14
tags: [docker, sysops-infra, module-14, labs, hands-on]
---

# 05 - Labs: Docker Compose

Môi trường: VM Linux đã cài Docker Engine 29.x kèm Compose plugin (`docker compose version` để xác nhận nhánh 5.x). Lưu file compose.yaml của từng lab vào [[labs/README|labs/]].

## Lab 1 (Basic): Từ `docker run` thủ công sang `compose.yaml`

1. Dùng lại kịch bản 4 lệnh `docker run` thủ công ở [[01-Introduction]] (nginx + backend + db + redis).
2. Viết `compose.yaml` tương đương, chạy `docker compose up -d`.
3. Xác nhận `docker compose ps` hiển thị đủ 4 service đang chạy.
4. Xác nhận network tự động tạo: `docker network ls | grep $(basename $(pwd))`.
5. `docker compose down`, xác nhận toàn bộ container và network bị xóa nhưng named volume vẫn còn (`docker volume ls`).

## Lab 2 (Basic): `depends_on` short syntax vs `condition: service_healthy`

1. Viết compose.yaml với `backend` phụ thuộc `db` bằng short syntax (`depends_on: [db]`), `db` là Postgres không có `healthcheck`.
2. Chạy `docker compose up -d`, quan sát log `backend` — nếu backend có logic kết nối DB ngay lúc khởi động, có khả năng gặp lỗi connection refused thoáng qua (tùy tốc độ máy).
3. Sửa lại: thêm `healthcheck` cho `db`, đổi `depends_on` sang long syntax `condition: service_healthy`.
4. Xóa toàn bộ (`docker compose down -v`), chạy lại từ đầu, quan sát: `docker compose ps` cho thấy `backend` chỉ chuyển sang `Created`/`Running` sau khi `db` đạt `healthy`.
5. Ghi lại bằng chứng (log timestamp) chứng minh thứ tự chờ đúng như khai báo.

## Lab 3 (Intermediate): Network tách biệt frontend/backend

1. Viết compose.yaml với 2 network riêng: `frontend` (chỉ `nginx` + `backend`) và `backend` (chỉ `backend` + `db`).
2. Xác nhận `nginx` không thể `ping`/kết nối trực tiếp tới `db`: `docker compose exec nginx ping -c 2 db` phải thất bại (không resolve được tên vì không cùng network).
3. Xác nhận `backend` kết nối được cả hai phía: `docker compose exec backend ping -c 2 nginx` và `docker compose exec backend ping -c 2 db` đều thành công.
4. Giải thích bằng ngôn ngữ của bạn: đây là mô hình bảo mật gì, vì sao hữu ích trong thực tế doanh nghiệp.

## Lab 4 (Intermediate): `.env`, biến môi trường, thứ tự ưu tiên

1. Tạo file `.env` khai báo `NGINX_PORT=8080`, dùng `${NGINX_PORT}` trong `ports:` của compose.yaml.
2. Chạy `docker compose up -d`, xác nhận `curl localhost:8080` hoạt động đúng theo biến từ `.env`.
3. Override tạm bằng biến shell: `NGINX_PORT=9090 docker compose up -d` — xác nhận biến shell thắng biến trong `.env` (đúng thứ tự ưu tiên đã học ở [[02-Theory]] mục 7.3).
4. Thử `docker compose config` để xem giá trị cuối cùng đã được nội suy — đối chiếu đúng với thứ tự ưu tiên.

## Lab 5 (Intermediate): Secrets — không hardcode password

1. Viết compose.yaml dùng file-based secret cho `POSTGRES_PASSWORD_FILE` như ví dụ ở [[04-Commands]] mục 4.
2. Chạy `docker compose up -d`, xác nhận `docker compose exec db cat /run/secrets/db_password` đọc đúng nội dung file.
3. Chạy `docker inspect <container-db>` tìm trong `Env` — xác nhận **không** thấy password dạng plaintext ở đó (chỉ thấy đường dẫn `_FILE`, không phải giá trị thật).
4. So sánh với cách làm sai (hardcode `POSTGRES_PASSWORD: mypassword123` trực tiếp trong `environment:`) — chạy thử, dùng `docker inspect` để chứng minh cách sai này lộ password ngay trong metadata container.

## Lab 6 (Advanced/Mini Project): Profiles cho môi trường debug

1. Mở rộng compose.yaml Lab 5, thêm service `adminer` (hoặc `pgadmin`) với `profiles: ["debug"]`.
2. Chạy `docker compose up -d` bình thường — xác nhận `adminer` không chạy.
3. Chạy `docker compose --profile debug up -d` — xác nhận `adminer` chạy thêm, truy cập được qua trình duyệt/`curl`.
4. Thử kích hoạt bằng biến môi trường thay vì flag: `COMPOSE_PROFILES=debug docker compose up -d`.
5. Viết ghi chú (không phải câu hỏi phỏng vấn) giải thích: trong quy trình CI/CD thực tế, biến `COMPOSE_PROFILES` có thể được set khác nhau giữa môi trường dev/staging/production như thế nào để tránh việc công cụ debug vô tình chạy trên production.

## Lab 7 (Advanced/Mini Project tổng hợp): Stack hoàn chỉnh production-grade

**Mục tiêu**: tổng hợp toàn bộ kiến thức module thành một compose.yaml đạt chuẩn review được trong doanh nghiệp — đây là bước đệm trực tiếp cho Module 15.

**Yêu cầu**: viết compose.yaml cho stack nginx + backend (tự build) + postgres + redis thỏa mãn TẤT CẢ:
- Không dùng khóa `version:`.
- `healthcheck` đầy đủ cho `db` và `redis`, `depends_on` dùng `condition: service_healthy` đúng chỗ.
- Network tách biệt `frontend`/`backend`, `db` không nằm trong `frontend`.
- Named volume cho `db_data`, bind mount `:ro` cho file cấu hình `nginx.conf`.
- Secret file-based cho mọi password, không hardcode trong `environment:`.
- `.env` cho các giá trị không nhạy cảm (port, tag image).
- `profiles: ["debug"]` cho ít nhất 1 service phụ trợ (admin UI).
- `restart: unless-stopped` cho mọi service chạy dài hạn.

Test toàn bộ bằng `docker compose config` (không lỗi schema), `docker compose up -d` (mọi service lên đúng thứ tự), `docker compose down -v` rồi `up -d` lại (dữ liệu Postgres mất — kỳ vọng vì `-v` xóa volume — dùng để kiểm chứng bạn hiểu rõ hệ quả của `-v`).

**Sản phẩm nộp**: compose.yaml hoàn chỉnh, `.env`, thư mục `secrets/` (thêm `.gitignore` loại trừ nội dung thật), lưu vào [[labs/README|labs/]].
