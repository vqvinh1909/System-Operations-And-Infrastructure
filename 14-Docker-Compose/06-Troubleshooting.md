---
title: "06 - Troubleshooting"
module: 14
tags: [docker, sysops-infra, module-14, troubleshooting]
---

# 06 - Troubleshooting: Docker Compose

## Sự cố 1: Sửa `compose.yaml` xong `docker compose up -d` nhưng container không đổi gì

**Nguyên nhân gốc**: Compose so sánh hash cấu hình (label `com.docker.compose.config-hash`) để quyết định có cần recreate container hay không. Một số thay đổi (ví dụ chỉ sửa **nội dung** file được bind mount, không đổi đường dẫn mount trong YAML) không làm thay đổi hash cấu hình — Compose "tưởng" không có gì đổi.

**Chẩn đoán**:
```bash
docker compose config          # xac nhan cau hinh moi da duoc doc dung
docker inspect <container> --format '{{index .Config.Labels "com.docker.compose.config-hash"}}'
```

**Khắc phục**: dùng `docker compose up -d --force-recreate` để buộc tạo lại container bất kể hash, hoặc với thay đổi nội dung file bind mount thuần túy (không đổi hành vi service), đôi khi chỉ cần restart tiến trình bên trong container là đủ (`docker compose restart <service>`) nếu ứng dụng tự đọc lại file cấu hình khi restart.

## Sự cố 2: `backend` liên tục crash loop lúc `docker compose up -d` lần đầu, nhưng chạy `docker compose restart backend` sau đó lại ổn

**Nguyên nhân gốc**: `depends_on` dùng short syntax (`condition: service_started` mặc định) — chỉ đảm bảo `db` container được start, không đảm bảo tiến trình Postgres bên trong đã sẵn sàng nhận kết nối. `backend` cố kết nối ngay khi start, gặp "connection refused", crash theo `restart` policy tạo cảm giác crash loop; sau vài giây database đã sẵn sàng, lần restart tiếp theo (do policy tự động hoặc thủ công) thành công.

**Khắc phục**: thêm `healthcheck` cho `db`, đổi `depends_on` sang long syntax với `condition: service_healthy` — Compose sẽ thực sự chờ `db` healthy trước khi tạo `backend`, loại bỏ crash loop ngay từ lần chạy đầu tiên.

## Sự cố 3: Biến trong `.env` không được nội suy vào `compose.yaml`

**Chẩn đoán**:
```bash
docker compose config | grep <TEN_BIEN>
ls -la .env    # xac nhan file .env ton tai DUNG thu muc project
```

**Nguyên nhân thường gặp**: file `.env` đặt sai vị trí (ví dụ trong thư mục con `config/.env` thay vì cùng cấp với `compose.yaml`), hoặc chạy lệnh Compose từ một thư mục khác (không phải project directory) mà không dùng `--project-directory`/`--env-file` chỉ định tường minh.

**Khắc phục**: di chuyển `.env` về đúng thư mục chứa `compose.yaml`, hoặc chỉ định tường minh bằng `docker compose --env-file <đường-dẫn> up -d`.

## Sự cố 4: `docker compose down -v` chạy nhầm trên production, mất dữ liệu database

**Nguyên nhân**: nhầm lẫn giữa `docker compose down` (chỉ xóa container + network, giữ nguyên named volume) và `docker compose down -v` (xóa cả named volume) — đặc biệt dễ xảy ra khi copy lệnh từ script dọn dẹp môi trường dev sang chạy nhầm trên production.

**Phòng ngừa**: tách biệt rõ ràng script/alias cho môi trường dev (được phép `-v` để làm sạch nhanh) và production (tuyệt đối không có `-v` trong bất kỳ script tự động nào chạm tới production); luôn backup volume theo quy trình đã học ở Module 13 trước khi chạy bất kỳ lệnh `down` nào trên môi trường có dữ liệu quan trọng, dù chỉ để "restart nhanh".

## Sự cố 5: `docker compose up` báo lỗi port đã bị chiếm, dù không thấy container nào đang publish port đó

**Chẩn đoán**:
```bash
sudo ss -tlnp | grep <port>
docker ps -a --filter "publish=<port>"
```

**Nguyên nhân**: một project Compose khác (thư mục khác, project name khác) hoặc một tiến trình ngoài Docker đang giữ port đó — `docker compose ps` chỉ hiện container thuộc project hiện tại, dễ khiến người debug bỏ sót nguồn xung đột thực sự.

**Khắc phục**: xác định đúng thủ phạm bằng `ss -tlnp` (không giới hạn phạm vi Docker), dừng đúng project/tiến trình đang giữ port, hoặc đổi port trong `.env`/`compose.yaml` của project hiện tại.

## Sự cố 6: Hai project Compose khác nhau vô tình dùng chung network do đặt tên trùng

**Triệu chứng**: container ở project A bất ngờ resolve được tên container ở project B, hoặc `docker compose up` báo lỗi network đã tồn tại với cấu hình khác.

**Nguyên nhân**: cả hai project vô tình có cùng project name (ví dụ cả hai thư mục đều tên `app`, hoặc cả hai đều khai báo cùng `name: app` tường minh trong compose.yaml) — Compose tiền tố network theo project name, hai project trùng tên sẽ đụng network mặc định `app_default`.

**Khắc phục**: đặt `name:` tường minh, duy nhất cho từng project trong compose.yaml (không phụ thuộc tên thư mục dễ trùng lặp), đặc biệt quan trọng khi nhiều team/nhiều project cùng chạy trên một server dùng chung.
