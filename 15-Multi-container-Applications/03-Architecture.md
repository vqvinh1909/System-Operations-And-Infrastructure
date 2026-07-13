---
title: "03 - Architecture"
module: 15
tags: [docker, sysops-infra, module-15, architecture, diagram]
---

# 03 - Architecture: Kiến trúc Stack, Luồng Request, Phân vùng Network

Toàn bộ sơ đồ đã được đo bằng script Python (so sánh `len()` border trên/dưới và mọi dòng nội dung trong từng khung) trước khi đưa vào file.

## 1. Kiến trúc tổng quan 5 thành phần

```text
+---------------------------------------------------------------+
|            KIEN TRUC STACK HOAN CHINH (MODULE 15)             |
+---------------------------------------------------------------+
| Client -> Nginx Reverse Proxy (port 80/443, publish ra ngoai) |
+---------------------------------------------------------------+
|          Nginx -> Web/App Backend (network frontend)          |
+---------------------------------------------------------------+
|  Backend -> MySQL/Postgres (du lieu ben vung, named volume)   |
+---------------------------------------------------------------+
|               Backend -> Redis (cache, session)               |
+---------------------------------------------------------------+
|    Backend -> RabbitMQ (message queue, xu ly bat dong bo)     |
+---------------------------------------------------------------+
|    Tat ca DB/Redis/RabbitMQ chi nam trong network backend     |
+---------------------------------------------------------------+
```

## 2. Luồng một request thực tế đi qua toàn bộ stack

```text
+----------------------------------------------------------------+
|              Client goi https://shop.example.com               |
+----------------------------------------------------------------+
                                  |
                                  v
+----------------------------------------------------------------+
|        Nginx Reverse Proxy nhan request tai port 443/80        |
+----------------------------------------------------------------+
                                  |
                                  v
+----------------------------------------------------------------+
| Nginx forward request toi backend qua ten service (DNS noi bo) |
+----------------------------------------------------------------+
                                  |
                                  v
+----------------------------------------------------------------+
|      Backend doc/ghi cache qua Redis (session, hot data)       |
+----------------------------------------------------------------+
                                  |
                                  v
+----------------------------------------------------------------+
|      Backend doc/ghi du lieu ben vung qua MySQL/Postgres       |
+----------------------------------------------------------------+
                                  |
                                  v
+----------------------------------------------------------------+
|       Backend day message xu ly nen (async) vao RabbitMQ       |
+----------------------------------------------------------------+
                                  |
                                  v
+----------------------------------------------------------------+
|      Worker rieng tieu thu message tu RabbitMQ, xu ly nen      |
+----------------------------------------------------------------+
```

**Đọc sơ đồ**: bốn bước đầu (Client → Nginx → Backend → Redis/Database) diễn ra **đồng bộ** trong đúng vòng đời của một HTTP request — client chờ tới khi có response. Bước cuối (RabbitMQ → Worker) diễn ra **bất đồng bộ**, hoàn toàn tách rời khỏi thời gian phản hồi client — đây chính là điểm khác biệt cốt lõi giữa Redis (nằm trong luồng đồng bộ, phục vụ tốc độ đọc) và RabbitMQ (nằm ngoài luồng đồng bộ, phục vụ tách rời tác vụ chậm) đã phân tích ở [[02-Theory]] mục 5.

## 3. Phân vùng network — nguyên tắc bảo mật cốt lõi

```text
+------------------------------------------------------------+
|                PHAN VUNG NETWORK CUA STACK                 |
+------------------------------------------------------------+
|                  frontend: nginx, backend                  |
+------------------------------------------------------------+
|         backend:  backend, mysql, redis, rabbitmq          |
+------------------------------------------------------------+
|  Chi nginx duoc publish port ra host (-p 80:80 / 443:443)  |
+------------------------------------------------------------+
```

**Nguyên tắc thực chiến**: chỉ có `backend` xuất hiện ở cả hai network (`frontend` và `backend`) — đóng vai trò "cầu nối" duy nhất. `nginx` không bao giờ nằm trong network `backend`, nên dù `nginx` có bị compromise (lỗ hổng, misconfiguration), kẻ tấn công vẫn không thể route thẳng tới database/Redis/RabbitMQ — phải đi xuyên qua đúng logic của `backend`. Đây là ứng dụng trực tiếp của mô hình network tách biệt đã học ở Module 12 và Module 14, áp dụng cho một stack thật với đầy đủ 5 thành phần thay vì ví dụ tối giản.

Tiếp theo: [[04-Commands]] để xem `compose.yaml` hoàn chỉnh hiện thực hóa toàn bộ 3 sơ đồ trên.
