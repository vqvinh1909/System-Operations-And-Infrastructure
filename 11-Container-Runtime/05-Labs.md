---
title: "05 - Labs"
module: 11
tags: [docker, sysops-infra, module-11, labs, hands-on]
---

# 05 - Labs: Container Runtime

Môi trường: VM Linux đã cài Docker Engine 29.x. Lưu kết quả vào [[labs/README|labs/]].

## Lab 1 (Basic): Vòng đời container và khác biệt `stop`/`kill`

1. Chạy `docker run -d --name lab-nginx nginx:1.27-alpine`, quan sát trạng thái bằng `docker ps` và `docker inspect --format '{{.State.Status}}' lab-nginx`.
2. Đo thời gian: `time docker stop lab-nginx` — ghi lại số giây thực tế.
3. Khởi động lại: `docker start lab-nginx`, rồi đo `time docker kill lab-nginx` — so sánh thời gian với bước 2.
4. Giải thích chênh lệch dựa trên cơ chế SIGTERM + grace period vs SIGKILL ngay lập tức.
5. Dọn dẹp: `docker rm lab-nginx`.

## Lab 2 (Basic): Tự viết ứng dụng bắt SIGTERM, so sánh có/không `--init`

1. Viết `worker.py`: một vòng lặp fork tiến trình con ngắn hạn (dùng `subprocess.Popen` gọi `sleep 1` liên tục), không tự implement reap.
2. Build image tối giản chạy `worker.py` làm entrypoint (dùng bind mount hoặc build image tạm — kỹ thuật Dockerfile đã học Module 10).
3. Chạy container KHÔNG có `--init`, để chạy vài phút, sau đó `docker exec <container> ps aux` — đếm số tiến trình ở trạng thái `Z` (zombie).
4. Chạy container CÓ `--init`, lặp lại quan sát tương tự — xác nhận không có zombie tích lũy.
5. Ghi lại bảng so sánh số lượng zombie theo thời gian giữa hai trường hợp.

## Lab 3 (Intermediate): Resource limit và OOM Kill

1. Viết một script Python cấp phát bộ nhớ tăng dần (`data = []; while True: data.append(bytearray(10*1024*1024)); time.sleep(0.1)`).
2. Chạy container với `--memory=100m --memory-swap=100m`, mount script vào và chạy nó.
3. Theo dõi bằng `docker stats --no-stream <container>` lặp lại vài lần để thấy memory usage tăng dần tới gần 100m.
4. Sau khi container bị kill, kiểm tra: `docker inspect <container> --format '{{.State.OOMKilled}} {{.State.ExitCode}}'`.
5. Kiểm tra `dmesg | grep -i "killed process"` trên host để thấy bằng chứng kernel-level.
6. Lặp lại với `--memory=100m` nhưng không set `--memory-swap` — quan sát sự khác biệt hành vi (container có thể sống lâu hơn nếu host có swap khả dụng).

## Lab 4 (Intermediate): Restart Policy và Restart Loop có kiểm soát

1. Viết một script luôn thoát với exit code 1 sau 2 giây (`sleep 2; exit 1`).
2. Chạy container với `--restart=on-failure:3`, theo dõi bằng `docker events --filter 'container=<tên>'` ở một terminal khác.
3. Đếm số lần container restart trước khi Docker dừng hẳn việc thử lại — xác nhận đúng bằng `:3` đã cấu hình.
4. Kiểm tra `docker inspect <container> --format '{{.RestartCount}}'`.
5. Lặp lại với `--restart=always` cho cùng script — quan sát: container restart vô hạn, không dừng lại như `on-failure:3`.
6. Chủ động `docker stop` container đang dùng `--restart=unless-stopped`, sau đó `sudo systemctl restart docker` — xác nhận container KHÔNG tự khởi động lại (khác với `always`).

## Lab 5 (Intermediate): Log rotation, tránh disk đầy

1. Chạy container ghi log liên tục không giới hạn: `docker run -d --name loggy busybox sh -c "while true; do echo $(date) - log line; sleep 0.05; done"`.
2. Để chạy vài phút, kiểm tra dung lượng file log thật: `docker inspect loggy --format '{{.LogPath}}'` rồi `ls -lh <đường dẫn đó>`.
3. Dừng và xóa container, chạy lại với `--log-opt max-size=1m --log-opt max-file=2`, lặp lại quan sát — xác nhận file log không vượt quá giới hạn cấu hình dù để chạy lâu hơn nhiều.
4. Cấu hình giới hạn log mặc định ở cấp daemon (`/etc/docker/daemon.json`), restart Docker, tạo container mới không chỉ định `--log-opt` — xác nhận nó vẫn tự động áp dụng giới hạn từ cấu hình daemon.

## Lab 6 (Advanced/Mini Project): Bộ dashboard giám sát bằng lệnh CLI thuần

**Mục tiêu**: tổng hợp `docker events`, `docker stats`, `docker system df` thành một script giám sát đơn giản.

1. Viết script bash chạy `docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"` định kỳ mỗi 30 giây, ghi log ra file.
2. Thêm phần script chạy `docker system df` định kỳ mỗi giờ, cảnh báo (in ra màn hình bằng màu/ký hiệu nổi bật) nếu tổng dung lượng build cache vượt một ngưỡng bạn tự chọn (ví dụ 5GB).
3. Thêm một tiến trình nền chạy `docker events --filter 'event=die' --filter 'event=oom'`, mỗi khi bắt được sự kiện, ghi log kèm timestamp và tên container.
4. Test toàn bộ script bằng cách cố tình tạo tình huống: một container bị OOM kill (dùng lại Lab 3), một container restart loop (dùng lại Lab 4) — xác nhận script phát hiện và ghi log đúng cả hai sự kiện.
5. Lưu script hoàn chỉnh vào `labs/mini-dashboard.sh`, kèm ghi chú giải thích từng phần.

**Kết quả cần đạt**: một script thực sự chạy được, chứng minh bạn hiểu và kết hợp thành thạo 4 công cụ vận hành cốt lõi: `docker events`, `docker stats`, `docker system df`, và exit code/OOM inspection.
