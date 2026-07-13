---
title: "06 - Troubleshooting"
module: 17
tags: [docker, sysops-infra, module-17, logging, monitoring, troubleshooting]
---

# 06 — Troubleshooting

## Sự cố 1: Disk đầy vì log không rotate

**Triệu chứng:** container/daemon Docker báo lỗi ghi (`no space left on device`), server chậm bất thường, nhiều dịch vụ khác cũng lỗi dù không liên quan tới container đang có vấn đề.

**Chẩn đoán:**
```bash
df -h /var/lib/docker
du -sh /var/lib/docker/containers/*/*-json.log 2>/dev/null | sort -rh | head -5
```

**Nguyên nhân gốc:** log driver `json-file` không được cấu hình `max-size`/`max-file`, kết hợp một container log liên tục (thường do vòng lặp lỗi hoặc log debug quên tắt).

**Khắc phục ngay (khẩn cấp):**
```bash
# Cat rong file log dang qua lon (KHONG xoa file, chi lam rong noi dung -
# xoa file se lam Docker mat tham chieu, container co the loi khi ghi tiep)
truncate -s 0 /var/lib/docker/containers/<container-id>/<container-id>-json.log
```

**Khắc phục lâu dài:** cấu hình `log-opts` trong `daemon.json` như ở [[04-Commands]], restart Docker, sau đó **recreate** (không chỉ restart) các container đang chạy để cấu hình mới có hiệu lực.

## Sự cố 2: cAdvisor báo lỗi hoặc không đọc được metric container

**Triệu chứng:** truy cập `http://<host>:8080/metrics` không thấy dữ liệu container nào, hoặc log cAdvisor báo lỗi liên quan `permission denied` khi đọc `/sys/fs/cgroup`.

**Chẩn đoán:**
```bash
docker logs cadvisor
```

**Nguyên nhân thường gặp:**
- Thiếu mount `/sys`, `/var/run`, `/var/lib/docker` như đã khai báo ở [[04-Commands]] — cAdvisor không có đường vào để đọc dữ liệu cgroup.
- Không chạy `privileged: true` (hoặc thiếu capability tương đương) trên host có cấu hình cgroup v2 nghiêm ngặt.
- Chạy trên môi trường cgroup v1/v2 khác với những gì image cAdvisor đang dùng hỗ trợ tốt nhất — cần kiểm tra version cAdvisor tương thích với kernel/cgroup driver của host.

**Khắc phục:** đối chiếu đúng danh sách volume mount trong docs chính thức, đảm bảo version cAdvisor đang dùng hỗ trợ cgroup v2 nếu host chạy cgroup v2 (`stat -fc %T /sys/fs/cgroup/` trả về `cgroup2fs` nghĩa là host đang dùng cgroup v2).

## Sự cố 3: Prometheus báo target `cadvisor` ở trạng thái `DOWN`

**Triệu chứng:** vào `http://<host>:9090/targets`, thấy target cAdvisor màu đỏ, dashboard Grafana không có dữ liệu (hoặc dữ liệu dừng lại từ một thời điểm).

**Chẩn đoán:**
1. Kiểm tra Prometheus có gọi được cAdvisor không (chạy lệnh từ chính container Prometheus, vì tên service DNS chỉ resolve được trong cùng Docker network):
   ```bash
   docker exec prometheus wget -qO- http://cadvisor:8080/metrics | head -5
   ```
2. Nếu lệnh trên lỗi `Name or service not known` → hai container không nằm chung một Docker network (kiểm tra `docker network inspect`).
3. Nếu lệnh trên lỗi `connection refused` → cAdvisor còn đang khởi động hoặc đã crash, kiểm tra lại Sự cố 2.
4. Nếu lệnh trên chạy được nhưng Prometheus vẫn báo `DOWN` → kiểm tra đúng `job_name`/`targets` trong `prometheus.yml` có khớp đúng tên service không, và Prometheus đã được restart/reload sau khi sửa file chưa (`docker exec prometheus kill -HUP 1` để reload config mà không cần restart toàn bộ container).

## Sự cố 4: Grafana không kết nối được Datasource (Prometheus/Loki)

**Triệu chứng:** vào **Configuration → Data Sources → Test**, Grafana báo lỗi `dial tcp: lookup prometheus: no such host` hoặc tương tự.

**Nguyên nhân gốc phổ biến nhất:** URL datasource khai báo `http://localhost:9090` thay vì `http://prometheus:9090`. Bên trong container Grafana, `localhost` chỉ trỏ về chính container Grafana, không phải host máy thật và cũng không phải container Prometheus — đây là nhầm lẫn rất phổ biến với người mới làm quen network của Docker Compose.

**Khắc phục:** luôn dùng đúng **tên service** đã khai báo trong `docker-compose.yml` làm hostname khi hai service giao tiếp nội bộ trong cùng network (Docker Compose tự tạo DNS nội bộ ánh xạ tên service → IP container).

## Sự cố 5: LogQL không trả về kết quả dù chắc chắn container đang có log

**Triệu chứng:** truy vấn `{container="payment-service"}` trong Grafana Explore không trả về gì, dù `docker logs payment-service` vẫn thấy log bình thường.

**Chẩn đoán và nguyên nhân thường gặp:**
1. **Sai tên label** — Alloy có thể gắn label khác với giá trị bạn nghĩ (ví dụ `container_name` thay vì `container`, tùy version/cấu hình). Vào Grafana Explore, dùng nút gợi ý label để xem chính xác label nào đang tồn tại thay vì đoán.
2. **Alloy chưa thấy container này** — kiểm tra `discovery.docker` có đúng trỏ tới `unix:///var/run/docker.sock` đã được mount vào container Alloy chưa (`docker exec alloy ls -la /var/run/docker.sock`).
3. **Khung thời gian truy vấn (time range) trong Grafana đang để sai** — lỗi rất phổ biến, đặc biệt khi máy lab và máy thật lệch múi giờ, khiến người học tưởng "không có log" trong khi log nằm ngoài khung thời gian đang xem.
4. **Loki từ chối log "out of order"** — Loki mặc định yêu cầu log trong cùng một stream (cùng tập label) phải đến theo đúng thứ tự thời gian; nếu nhiều container cùng gắn một label giống hệt nhau và log gửi lệch thứ tự, Loki có thể từ chối một phần dữ liệu. Kiểm tra log của chính container Loki (`docker logs loki`) để thấy lỗi `entry out of order` nếu có, và cân nhắc gắn thêm label phân biệt rõ hơn theo container/instance.

## Sự cố 6: Alert Grafana không bao giờ chuyển sang `Firing`

**Triệu chứng:** điều kiện rõ ràng đã đúng (CPU quan sát bằng mắt trên dashboard đã vượt ngưỡng), nhưng alert vẫn ở trạng thái `Pending` mãi không `Firing`, hoặc không chuyển trạng thái nào cả.

**Nguyên nhân thường gặp:**
- Thời gian `for` (khoảng thời gian điều kiện phải đúng liên tục trước khi Firing) đặt quá dài so với thời gian test thực tế trong lab.
- Truy vấn PromQL trong rule khác với truy vấn bạn đang nhìn trên dashboard (dễ gõ sai một ký tự khi copy sang ô cấu hình alert).
- Notification channel (email/Slack/webhook) cấu hình sai khiến bạn tưởng alert "không hoạt động", trong khi thực ra alert đã Firing nhưng không có ai/kênh nào nhận được thông báo — luôn kiểm tra trạng thái alert ngay trong Grafana UI trước khi kết luận alert không hoạt động.
