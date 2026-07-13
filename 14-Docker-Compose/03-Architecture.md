---
title: "03 - Architecture"
module: 14
tags: [docker, sysops-infra, module-14, architecture, diagram]
---

# 03 - Architecture: Project Naming, Luồng `docker compose up`

Toàn bộ sơ đồ đã được đo bằng script Python (so sánh `len()` border trên/dưới và mọi dòng nội dung trong từng khung) trước khi đưa vào file.

## 1. Project name — tiền tố của mọi resource

```text
+------------------------------------------------------------+
|          PROJECT NAME LA TIEN TO CUA MOI RESOURCE          |
+------------------------------------------------------------+
| Thu muc myshop/ chua compose.yaml -> project name = myshop |
+------------------------------------------------------------+
|  Container: myshop-db-1, myshop-backend-1, myshop-nginx-1  |
+------------------------------------------------------------+
|              Network mac dinh: myshop_default              |
+------------------------------------------------------------+
|                   Volume: myshop_db_data                   |
+------------------------------------------------------------+
```

Đây là lý do hai project Compose độc lập (hai thư mục khác nhau, mỗi thư mục một `compose.yaml`) không bao giờ đụng nhau dù dùng chung tên service — Compose luôn tiền tố mọi resource thật bằng project name, dù trong file YAML bạn chỉ viết tên service ngắn gọn (`db`, `backend`, `nginx`).

## 2. Luồng `docker compose up -d` theo đúng thứ tự nội bộ

```text
+--------------------------------------------------------+
|             docker compose up -d duoc goi              |
+--------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------+
|    Doc + merge compose.yaml, nap .env, noi suy bien    |
+--------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------+
| Validate theo Compose Specification (schema moi nhat)  |
+--------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------+
|          Tinh dependency graph tu depends_on           |
+--------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------+
|                Tao network chua ton tai                |
+--------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------+
|                Tao volume chua ton tai                 |
+--------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------+
|    db: tao/start container, cho healthcheck healthy    |
+--------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------+
| backend: cho db healthy roi moi tao/start (depends_on) |
+--------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------+
|    nginx: cho backend service_started roi tao/start    |
+--------------------------------------------------------+
```

**Đọc sơ đồ**: thứ tự network/volume luôn được tạo **trước** bất kỳ container nào, vì container cần các resource đó tồn tại sẵn để gắn vào lúc start. Thứ tự start container giữa các service tuân theo dependency graph tính từ `depends_on` — `db` luôn đi trước `backend`, `backend` luôn đi trước `nginx` trong ví dụ này. Với `condition: service_healthy` (như `db`), Compose thực sự **chờ** (poll theo `interval` của healthcheck) cho tới khi container đạt trạng thái `healthy` mới tiếp tục bước kế tiếp — khác với `service_started` (mặc định của short syntax `depends_on`) chỉ chờ container được start, không quan tâm tiến trình bên trong đã sẵn sàng phục vụ hay chưa.

Tiếp theo: [[04-Commands]] để xem cú pháp lệnh cụ thể và file `compose.yaml` mẫu đầy đủ.
