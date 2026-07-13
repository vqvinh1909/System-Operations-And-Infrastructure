---
title: "03 - Architecture"
module: 7
tags: [ansible, sysops-infra, module-07, architecture, diagram, multi-tier, zero-downtime]
---

# 03 — Kiến trúc & Sơ đồ

## Sơ đồ 1: Kiến trúc Multi-tier tổng thể

```
+----------------------+
|   INTERNET / USER    |
+----------------------+
           |
           v
+----------------------------------+
|        LOAD BALANCER TIER        |
|   HAProxy / Nginx (2 node HA)    |
+----------------------------------+
           |
           v
+----------------------------------+
|          WEB / APP TIER          |
|         web-01 .. web-20         |
+----------------------------------+
           |
           v
+----------------------------------+
|          DATABASE TIER           |
|       primary + 2 replica        |
+----------------------------------+
```

Đọc sơ đồ: traffic luôn đi qua Load Balancer trước khi tới Web/App tier — đây chính là điểm mấu chốt cho phép Zero Downtime Deployment, vì Load Balancer có thể "che giấu" việc một node Web tier đang tạm ngừng phục vụ để cập nhật. Database tier nằm sâu nhất, ít node nhất, và **không** nằm sau Load Balancer theo kiểu round-robin như Web tier — chỉ có primary nhận ghi, replica chủ yếu phục vụ đọc hoặc dự phòng.

## Sơ đồ 2: Rolling Update theo từng đợt (`serial`)

```
+----------------------------------+
|  Dot 1: serial batch 1 (2 host)  |
+----------------------------------+
                 |
                 v
+----------------------------------+
|  Dot 2: serial batch 2 (2 host)  |
+----------------------------------+
                 |
                 v
+----------------------------------+
| Dot N: serial batch N (con lai)  |
+----------------------------------+
```

Đọc sơ đồ: mỗi đợt (`serial` batch) được coi như một Play hoàn chỉnh độc lập — hoàn tất toàn bộ Task cho batch hiện tại trước khi bắt đầu batch kế tiếp. Nếu `max_fail_percentage` bị vượt ngưỡng ở bất kỳ đợt nào, các đợt sau **không chạy** — phần lớn hạ tầng vẫn giữ nguyên phiên bản cũ, ổn định.

## Sơ đồ 3: Chuỗi 6 bước Zero Downtime cho một node

```
+---------------------------------------------+
| Buoc 1: Go node khoi pool HAProxy (disable) |
+---------------------------------------------+
                      |
                      v
+---------------------------------------------+
|    Buoc 2: Cho drain ket noi dang xu ly     |
+---------------------------------------------+
                      |
                      v
+---------------------------------------------+
|     Buoc 3: Cap nhat ung dung tren node     |
+---------------------------------------------+
                      |
                      v
+---------------------------------------------+
|           Buoc 4: Restart service           |
+---------------------------------------------+
                      |
                      v
+---------------------------------------------+
|     Buoc 5: Health check (retry/until)      |
+---------------------------------------------+
                      |
                      v
+---------------------------------------------+
|   Buoc 6: Dua node tro lai pool (enable)    |
+---------------------------------------------+
```

Đọc sơ đồ: đây là chuỗi Task chạy cho **từng node một** (`serial: 1` hoặc theo batch), lặp lại tuần tự cho tới khi mọi node được cập nhật. Điểm sống còn là Bước 5 — nếu health check thất bại, Task ở [[02-Theory|02 - Theory]] mục 3 dùng `when: health_result.status == 200` để **không** thực hiện Bước 6, node bị lỗi bị bỏ lại ngoài pool thay vì được đưa trở lại phục vụ traffic thật trong tình trạng chưa sẵn sàng.
