---
title: "06 - Troubleshooting"
module: 9
tags: [docker, sysops-infra, module-09, troubleshooting]
---

# 06 - Troubleshooting: Docker Engine

## Sự cố 1: `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?`

**Chẩn đoán theo thứ tự**:
```bash
systemctl status docker         # daemon co dang chay khong
journalctl -u docker.service -n 100 --no-pager   # tim dong loi cuoi cung truoc khi crash
ls -l /var/run/docker.sock      # socket co ton tai khong
groups $USER                    # user hien tai co trong nhom docker khong
```

**Nguyên nhân thường gặp**:
- Daemon chưa start hoặc crash do lỗi cú pháp `daemon.json` (JSON không hợp lệ khiến `dockerd` không khởi động được — log `journalctl` sẽ báo rõ dòng lỗi parse).
- User hiện tại không thuộc nhóm `docker` và không dùng `sudo` — lỗi permission bị báo nhầm thành "daemon not running".
- Socket bị đổi đường dẫn do cấu hình tùy chỉnh (`-H` khác mặc định) mà biến môi trường `DOCKER_HOST` client không trỏ đúng.

**Khắc phục**: sửa `daemon.json` (dùng `python3 -m json.tool` kiểm tra cú pháp trước khi restart), hoặc thêm user vào nhóm `docker` đúng quy trình (đã cân nhắc rủi ro ở [[02-Theory]] mục 4), `newgrp docker` hoặc logout/login lại để nhóm có hiệu lực.

## Sự cố 2: Daemon start được nhưng container cũ "biến mất" sau khi đổi `storage-driver`

**Triệu chứng**: sau khi sửa `daemon.json` đổi `storage-driver` rồi restart, `docker ps -a` trống trơn dù trước đó có nhiều container.

**Nguyên nhân gốc**: mỗi storage driver (`overlay2`, `btrfs`, `zfs`...) lưu dữ liệu image/container ở cấu trúc thư mục khác nhau bên dưới `/var/lib/docker/`. Đổi driver không "migrate" dữ liệu cũ — Docker daemon đơn giản là **không còn thấy** dữ liệu được lưu theo driver cũ nữa (dữ liệu vẫn còn trên đĩa, chỉ là daemon đang đọc nhầm chỗ).

**Khắc phục**: không đổi `storage-driver` tùy tiện trên node đang có dữ liệu quan trọng. Nếu bắt buộc đổi, backup container/image cần giữ (export image, backup volume) trước khi đổi, hoặc chấp nhận build lại từ đầu trên driver mới.

## Sự cố 3: Container liên tục exit với code 137

**Chẩn đoán**:
```bash
docker inspect <container> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
dmesg | grep -i "killed process"
```

**Nguyên nhân**: exit code 137 = 128 + 9 (SIGKILL). Hai khả năng chính:
1. `State.OOMKilled = true` — container chạm giới hạn `--memory` (cgroup, đã học Module 08), kernel OOM-killer giết tiến trình chính.
2. Bị `docker kill` hoặc healthcheck/orchestrator (Compose, Kubernetes) chủ động gửi SIGKILL sau khi healthcheck fail nhiều lần liên tiếp.

**Khắc phục**: nếu do OOM, tăng `--memory` hợp lý hoặc tìm memory leak trong ứng dụng; nếu do healthcheck, kiểm tra `docker inspect --format '{{json .State.Health}}'` để xem log các lần healthcheck fail trước đó.

## Sự cố 4: `docker run` treo rất lâu ở bước pull image, cuối cùng báo `429 Too Many Requests`

**Nguyên nhân gốc**: Docker Hub áp rate-limit cho pull ẩn danh (không login) theo IP — môi trường CI/CD dùng chung IP NAT (nhiều runner cùng ra Internet qua 1 IP) rất dễ chạm giới hạn này khi build nhiều lần trong ngày.

**Khắc phục**: `docker login` với tài khoản Docker Hub có hạn mức cao hơn trong pipeline CI, hoặc cấu hình `registry-mirrors` trỏ tới registry mirror nội bộ/pull-through cache trong `daemon.json`, hoặc host image base quan trọng trên private registry nội bộ để không phụ thuộc Docker Hub cho các lần build lặp lại.

## Sự cố 5: Bật TCP socket không TLS cho tiện quản trị từ xa, sau đó server bị chiếm quyền

**Triệu chứng thực tế đã xảy ra tại nhiều tổ chức**: kỹ sư mở `dockerd -H tcp://0.0.0.0:2375` để "cho tiện" gọi API từ xa lúc debug, quên tắt lại. Vài ngày sau server bị quét cổng, kẻ tấn công gọi thẳng Docker API (`POST /containers/create` với mount `/` vào container) để lấy root shell trên host — không cần mật khẩu, không cần khai thác lỗ hổng nào, chỉ cần cổng 2375 mở ra Internet.

**Khắc phục / phòng ngừa**:
- Không bao giờ bind Docker API ra TCP mà không có TLS (`--tlsverify` với client certificate) trong môi trường có thể truy cập từ mạng không tin cậy.
- Nếu cần quản trị từ xa, ưu tiên SSH tunnel (`docker -H ssh://user@host`) thay vì mở TCP port trực tiếp.
- Rà soát định kỳ `netstat -tlnp | grep 2375` hoặc dùng công cụ quét cổng nội bộ để phát hiện sớm cấu hình sai này trên toàn hạ tầng.

## Sự cố 6: `docker exec` vào container không thấy tiến trình healthcheck script vừa chạy xong

**Triệu chứng**: healthcheck script định nghĩa trong image chạy nhanh, khi `docker exec` vào kiểm tra thủ công thì lệnh đã kết thúc từ lâu — dễ gây hiểu lầm "healthcheck không chạy".

**Giải thích đúng bản chất**: mỗi lần Docker Engine chạy healthcheck (`HEALTHCHECK` trong Dockerfile hoặc `--health-cmd`), nó tạo một tiến trình **ngắn hạn mới** bên trong đúng namespace của container (giống cơ chế `docker exec`/`nsenter`), chạy xong thì thoát — không phải một tiến trình nền tồn tại liên tục để bạn `docker exec` vào "bắt gặp". Xem log/kết quả healthcheck đúng cách bằng `docker inspect --format '{{json .State.Health}}' <container>`, không phải bằng cách cố tìm tiến trình đang chạy.
