---
title: "06 - Troubleshooting"
module: 10
tags: [docker, sysops-infra, module-10, troubleshooting]
---

# 06 - Troubleshooting: Docker Images / Build

## Sự cố 1: Build local 20 giây, CI/CD mất 8 phút cho cùng Dockerfile

**Chẩn đoán**: xem output build có dòng `CACHED` không:
```bash
docker build -t myapp . 2>&1 | grep -c CACHED
```

**Nguyên nhân thường gặp**: CI/CD runner dùng container ephemeral (dùng xong xóa sạch), không giữ lại build cache giữa các lần chạy — mỗi lần build đều là `--no-cache` trên thực tế dù không khai báo cờ đó. Máy local giữ nguyên `/var/lib/docker` giữa các lần build nên luôn tận dụng được cache.

**Khắc phục**: cấu hình CI cache layer bằng `--cache-from`/`--cache-to` của buildx trỏ tới registry cache (`type=registry`), hoặc dùng cache mount cho dependency (không phụ thuộc layer cache), hoặc dùng runner có persistent volume cho `/var/lib/docker`.

## Sự cố 2: `COPY failed: file not found` dù file rõ ràng tồn tại trong thư mục

**Nguyên nhân gốc**: nhầm build context. Đường dẫn trong `COPY` luôn tính từ **build context** (thư mục truyền vào cuối lệnh `docker build <context>`), không phải từ vị trí Dockerfile. Nếu chạy `docker build -f subdir/Dockerfile .` nhưng file cần copy nằm trong `subdir/`, lệnh `COPY app.py .` sẽ tìm `app.py` ở context gốc (`.`), không tìm trong `subdir/`.

**Chẩn đoán**:
```bash
# Kiem tra dung file nao thuc su nam trong build context BuildKit nhin thay
docker build --no-cache --progress=plain -t debug . 2>&1 | grep "transferring context"
```

**Khắc phục**: xác định rõ context đúng, hoặc dùng đường dẫn tương đối đúng theo context, hoặc di chuyển Dockerfile ra thư mục gốc của context.

## Sự cố 3: Image phình to bất thường dù đã multi-stage

**Chẩn đoán**:
```bash
docker history --no-trunc <image>
docker images <image> --format "{{.Size}}"
```

**Nguyên nhân thường gặp**: quên `--mount=type=cache` hoặc quên dọn cache package manager trong **cùng RUN**, khiến stage builder nặng không thành vấn đề (không mang sang final) nhưng **stage final** vô tình cài thêm gói không cần thiết (ví dụ cài `build-essential` ở stage final để "cho chắc" thay vì chỉ ở stage builder).

**Khắc phục**: rà soát từng `RUN` ở stage final — chỉ được phép chứa runtime dependency (thư viện hệ thống cần khi chạy, ví dụ `ca-certificates`, `libssl`), không được có compiler/dev-header/source code.

## Sự cố 4: `no space left on device` trên build server dù ổ đĩa "còn trống" theo `df -h`

**Chẩn đoán**:
```bash
docker system df           # tong dung luong Docker dang chiem (images, containers, volumes, build cache)
docker system df -v        # chi tiet theo tung image/container/volume
```

**Nguyên nhân**: build cache (đặc biệt cache mount `--mount=type=cache`) và layer image cũ tích tụ theo thời gian trong `/var/lib/docker`, không tự động dọn. Trên build server chạy CI liên tục hàng trăm build/ngày, dung lượng này tăng rất nhanh.

**Khắc phục**:
```bash
docker builder prune --all --force     # don build cache
docker image prune -a --force          # don image khong con container nao dung
docker system prune -a --volumes --force   # don toan bo (can trong voi volume du lieu that)
```
Nên đặt lịch định kỳ (cron/CI scheduled job) chạy `docker builder prune` thay vì đợi tới khi hết dung lượng mới xử lý.

## Sự cố 5: Secret bị lộ trong image dù không có dòng nào `COPY secret.txt`

**Triệu chứng**: security scan báo image chứa API key, dù Dockerfile "nhìn qua" không copy file secret nào.

**Nguyên nhân gốc**: dùng `ARG`/`ENV` để truyền secret (`ARG NPM_TOKEN`, rồi `RUN npm config set ... $NPM_TOKEN`) — giá trị này **vẫn lưu trong build history/metadata** của image, xem được bằng `docker history --no-trunc` hoặc `docker inspect`, dù không có file nào bị `COPY` trực tiếp.

**Chẩn đoán**:
```bash
docker history --no-trunc <image> | grep -i token
```

**Khắc phục**: chuyển toàn bộ secret sang BuildKit secret mount (`--mount=type=secret`), rebuild lại image từ đầu (image cũ đã leak secret cần được **xóa khỏi registry** và **thu hồi (revoke) secret đó**, không chỉ sửa Dockerfile là đủ — secret đã từng nằm trong layer công khai coi như đã bị lộ).

## Sự cố 6: Container graceful shutdown không hoạt động dù đã set `STOPSIGNAL`

**Nguyên nhân thường gặp**: `CMD`/`ENTRYPOINT` dùng shell form (`CMD node server.js` thay vì `CMD ["node", "server.js"]`), khiến tiến trình ứng dụng chạy như con của `/bin/sh -c`, không phải PID 1. `STOPSIGNAL` được Docker gửi đúng tới PID 1 (là `/bin/sh`), nhưng `/bin/sh` không tự động forward signal cho tiến trình con — ứng dụng thực sự không bao giờ nhận được tín hiệu.

**Chẩn đoán**:
```bash
docker exec <container> ps aux
# Neu PID 1 la /bin/sh -c thay vi ung dung -> day la nguyen nhan
```

**Khắc phục**: đổi sang exec form (`CMD ["node", "server.js"]`), hoặc nếu bắt buộc dùng shell form vì cần xử lý biến môi trường/wildcard, thêm `exec` ở đầu lệnh shell (`CMD exec node server.js`) để `exec` thay thế tiến trình shell bằng chính tiến trình ứng dụng thay vì fork con mới.
