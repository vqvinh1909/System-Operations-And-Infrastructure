---
title: "06 - Troubleshooting"
module: 15
tags: [docker, sysops-infra, module-15, troubleshooting]
---

# 06 - Troubleshooting: Multi-container Applications

## Sự cố 1: Toàn bộ stack chậm bất thường, nghi ngờ database nhưng thực ra là Redis

**Chẩn đoán theo thứ tự**:
```bash
docker compose exec redis redis-cli info stats | grep -E "keyspace_hits|keyspace_misses"
docker compose logs backend | grep -i "cache miss" | tail -50
docker compose exec db psql -U postgres -d shopdb -c "SELECT count(*) FROM pg_stat_activity;"
```

**Nguyên nhân thường gặp**: Redis bị restart (mất toàn bộ cache trong RAM, đúng đặc điểm đã học ở [[02-Theory]] mục 4.3) hoặc TTL cấu hình quá ngắn khiến cache liên tục miss — mọi request đổ dồn thẳng vào database, database vẫn hoạt động bình thường nhưng chịu tải gấp nhiều lần thiết kế ban đầu, tạo cảm giác "database chậm" dù gốc rễ là cache không hoạt động.

**Khắc phục**: xác nhận tỷ lệ `keyspace_hits`/`keyspace_misses` của Redis, nếu tỷ lệ miss cao bất thường so với baseline, kiểm tra thời điểm Redis restart gần nhất (`docker compose ps` cột uptime), điều chỉnh TTL cache hợp lý hơn, cân nhắc bật Redis persistence nếu cache "khởi động lạnh" (cold start) sau mỗi lần restart gây tải đột biến không chấp nhận được cho database.

## Sự cố 2: Message trong RabbitMQ tồn đọng ngày càng nhiều, worker không xử lý kịp

**Chẩn đoán**:
```bash
docker compose --profile debug exec rabbitmq rabbitmq-diagnostics ping
curl -u shopapp:<password> http://localhost:15672/api/queues | python3 -m json.tool
docker compose logs worker --tail 100
```

**Nguyên nhân thường gặp**: (1) worker bị lỗi/crash liên tục, restart policy che giấu triệu chứng (tương tự bài học Module 11) — kiểm tra `docker compose ps` xem `RestartCount` của worker; (2) tốc độ tiêu thụ message chậm hơn tốc độ sinh ra (ví dụ mỗi message worker xử lý tốn 2 giây nhưng backend tạo ra 10 message/giây) — cần scale số lượng worker.

**Khắc phục**: nếu do lỗi worker, sửa lỗi và xác nhận queue giảm dần sau khi worker ổn định. Nếu do throughput không đủ, scale worker: `docker compose up -d --scale worker=3` (Compose tự tạo thêm 2 container worker nữa, tất cả cùng tiêu thụ chung một queue, RabbitMQ tự phân phối message theo cơ chế round-robin giữa các consumer đang kết nối).

## Sự cố 3: `docker compose up -d` lần đầu luôn có backend crash 1-2 lần trước khi ổn định

**Chẩn đoán**: xem `docker compose logs backend | head -50` để tìm lỗi kết nối cụ thể tới thành phần nào (db/redis/rabbitmq), đối chiếu thời điểm với `docker compose ps` xem thành phần đó đã `healthy` chưa tại thời điểm backend cố kết nối.

**Nguyên nhân**: một trong ba `healthcheck` (db/redis/rabbitmq) có `start_period` quá ngắn so với thời gian khởi tạo thật của nó trên máy lab cụ thể (máy chậm hơn máy dùng để test ban đầu), khiến healthcheck báo `unhealthy` oan trong lúc thành phần đó vẫn đang khởi tạo bình thường — nhưng vì `depends_on.condition: service_healthy` chờ đúng trạng thái `healthy`, backend vẫn phải đợi thêm vài chu kỳ `interval` trước khi thực sự có thể start, đây là hành vi đúng, không phải bug (chỉ chậm hơn dự kiến).

**Khắc phục**: tăng `start_period` cho thành phần khởi tạo chậm (đặc biệt RabbitMQ, thường mất 15-20 giây ở lần chạy đầu để khởi tạo Mnesia database nội bộ), hoặc chấp nhận độ trễ khởi động lần đầu là bình thường và không phải "lỗi" cần sửa nếu stack cuối cùng vẫn lên đúng và ổn định.

## Sự cố 4: Nginx trả về `502 Bad Gateway` dù `backend` đang `Up` và `healthy`

**Chẩn đoán**:
```bash
docker compose logs nginx --tail 50
docker compose exec nginx wget -qO- http://backend:3000/health
```

**Nguyên nhân thường gặp**: (1) `backend` healthy theo healthcheck riêng của nó (có thể chỉ kiểm tra kết nối database, không kiểm tra chính cổng HTTP mà Nginx đang cố kết nối tới) nhưng cổng HTTP thực sự chưa sẵn sàng lắng nghe (ví dụ ứng dụng bind port sau khi đã báo healthy do thứ tự code sai); (2) cấu hình `upstream`/`proxy_pass` trong `nginx.conf` trỏ sai tên service hoặc sai port so với cổng backend thực sự lắng nghe.

**Khắc phục**: đảm bảo healthcheck của `backend` thực sự kiểm tra đúng cổng HTTP mà Nginx sẽ gọi tới (không chỉ kiểm tra kết nối database bên trong), rà soát lại `nginx.conf` khớp đúng tên service và port.

## Sự cố 5: Backup database trong lúc stack đang chạy cho ra file backup không nhất quán

**Nguyên nhân**: nếu backend vẫn đang ghi dữ liệu liên tục trong lúc container tạm chạy `tar czf` để nén thư mục dữ liệu Postgres trực tiếp (không qua công cụ dump chuyên dụng), có rủi ro chụp được trạng thái file đang ghi dở (đặc biệt với write-ahead log của Postgres).

**Khắc phục đúng cho database production**: thay vì nén thẳng thư mục dữ liệu bằng `tar` (chỉ phù hợp khi container đã dừng hoặc chấp nhận rủi ro không nhất quán), dùng công cụ dump chuyên dụng của chính database (`pg_dump`/`pg_dumpall` cho Postgres, `mysqldump` cho MySQL) chạy qua `docker compose exec db pg_dump -U postgres shopdb > backup.sql` — các công cụ này đảm bảo tính nhất quán logic của dữ liệu ngay cả khi database đang phục vụ traffic, khác hẳn việc nén file vật lý thô.

## Sự cố 6: Scale `backend` lên nhiều instance nhưng người dùng bị "đăng xuất" ngẫu nhiên

**Nguyên nhân**: backend lưu session trong bộ nhớ của chính nó (vi phạm nguyên tắc stateless đã nhấn mạnh ở [[02-Theory]] mục 2) thay vì lưu vào Redis dùng chung — request lần sau của cùng người dùng có thể rơi vào một instance backend khác (do Nginx load balancing round-robin) không có session đó trong bộ nhớ, hệ thống coi như người dùng chưa đăng nhập.

**Khắc phục**: sửa logic ứng dụng chuyển toàn bộ session sang lưu ở Redis (session store dùng chung giữa mọi instance backend) — đây chính là lý do kiến trúc đã đưa Redis vào stack ngay từ đầu, không phải tùy chọn có cũng được không có cũng được.
