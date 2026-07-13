---
title: "05 - Labs"
module: 17
tags: [docker, sysops-infra, module-17, logging, monitoring, labs]
---

# 05 — Labs thực hành

## Lab 1: Cấu hình Log Rotation và tự tay làm đầy đĩa (có kiểm soát)

**Mục tiêu:** hiểu rõ hậu quả của việc không cấu hình rotation, bằng cách tự tạo ra tình huống đó trong môi trường lab an toàn.

1. Trên một máy lab (không phải production), chạy một container cố tình log liên tục không giới hạn:
   ```bash
   docker run -d --name log-flood busybox sh -c "while true; do echo 'log line filler filler filler'; done"
   ```
2. Theo dõi dung lượng file log tăng lên theo thời gian bằng lệnh `du -sh` đã học ở [[04-Commands]]. Quan sát tốc độ tăng.
3. Dừng container, xóa nó, cấu hình lại `daemon.json` với `max-size: 5m`, `max-file: 2`, restart Docker.
4. Chạy lại đúng container log-flood đó, quan sát file log lần này có tự rotate và giới hạn dung lượng đúng như cấu hình không.
5. Ghi vào `notes.md`: dung lượng log tăng bao nhiêu MB/phút khi không giới hạn, và thời gian ước tính để làm đầy một đĩa 20GB nếu để tình huống này chạy trên production thật.

## Lab 2: Dựng pipeline Metrics hoàn chỉnh (cAdvisor → Prometheus → Grafana)

1. Dùng file `docker-compose.yml` mẫu ở [[04-Commands]] (bỏ phần Loki/Alloy nếu muốn làm riêng từng phần), khởi động `cadvisor`, `prometheus`, `grafana`.
2. Mở `http://<host>:8080` kiểm tra cAdvisor đã lên, xem trực tiếp các container đang được theo dõi.
3. Mở `http://<host>:9090/targets` trên Prometheus, xác nhận target `cadvisor` ở trạng thái `UP`.
4. Đăng nhập Grafana (`http://<host>:3000`), thêm datasource Prometheus trỏ tới `http://prometheus:9090`.
5. Tạo một dashboard mới với ít nhất 3 panel: CPU theo container (dùng `rate(container_cpu_usage_seconds_total[5m])`), RAM theo container, và số lượng container đang chạy.
6. Chạy thêm 2-3 container tải nhẹ (ví dụ `docker run -d --name stress polinux/stress stress --cpu 1`) và quan sát dashboard cập nhật gần thời gian thực.

## Lab 3: Dựng pipeline Logs hoàn chỉnh (Grafana Alloy → Loki → Grafana)

1. Tạo file `config.alloy` theo mẫu ở [[04-Commands]], khởi động thêm `loki` và `alloy` trong cùng compose file.
2. Trong Grafana, thêm datasource Loki trỏ tới `http://loki:3100`.
3. Mở tab **Explore**, chọn datasource Loki, thử truy vấn `{container="payment-service"}` (thay bằng tên container thật bạn đang chạy) — xác nhận thấy log real-time.
4. Chạy một container cố tình in log lỗi định kỳ để luyện truy vấn LogQL:
   ```bash
   docker run -d --name error-generator busybox sh -c \
     "while true; do echo 'INFO: request handled'; sleep 2; echo 'ERROR: connection timeout'; sleep 8; done"
   ```
5. Viết truy vấn LogQL đếm số dòng `ERROR` trong 5 phút gần nhất, đối chiếu kết quả `count_over_time` với việc tự đếm bằng mắt qua `docker logs`.

## Lab 4: Dashboard hợp nhất và Alert cơ bản

1. Trên cùng một dashboard Grafana, thêm một panel kiểu **Logs** trỏ tới datasource Loki, lọc theo container đang bị theo dõi CPU cao.
2. Cấu hình một **Alert rule** trong Grafana: điều kiện CPU container vượt 80% liên tục trong 2 phút (dùng PromQL `rate(container_cpu_usage_seconds_total[2m]) > 0.8`).
3. Thử trigger alert bằng container stress ở Lab 2, quan sát trạng thái alert chuyển từ `Normal` → `Pending` → `Firing` trong Grafana UI.
4. Ghi lại vào `notes.md`: alert mất bao lâu để chuyển sang `Firing` kể từ lúc điều kiện thật sự đúng, và giải thích vì sao có độ trễ đó (liên hệ tới `scrape_interval` và khoảng thời gian đánh giá rule — `for` duration).

## Tiêu chí hoàn thành

- Dựng được toàn bộ 5 service (cAdvisor, Prometheus, Loki, Alloy, Grafana) chạy ổn định bằng một file Docker Compose duy nhất.
- Có ít nhất một dashboard Grafana hiển thị cả metric lẫn log của cùng một container.
- Viết được tối thiểu 3 truy vấn PromQL và 3 truy vấn LogQL khác với ví dụ mẫu trong tài liệu.
- Hiểu và giải thích lại được bằng lời của mình vì sao cần cấu hình log rotation ngay từ đầu, không chờ tới khi sự cố xảy ra mới sửa.
