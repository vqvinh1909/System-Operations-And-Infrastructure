---
title: "04 - Commands"
module: 17
tags: [docker, sysops-infra, module-17, logging, monitoring, commands]
---

# 04 — Lệnh, Cấu hình, Ví dụ mẫu

## Cấu hình Log Rotation cho `json-file` (bắt buộc trong production)

```bash
# Cau hinh mac dinh cho TOAN BO host - sua /etc/docker/daemon.json
cat /etc/docker/daemon.json
# {
#   "log-driver": "json-file",
#   "log-opts": {
#     "max-size": "10m",
#     "max-file": "3"
#   }
# }
sudo systemctl restart docker
```

- `max-size: 10m`: mỗi file log tối đa 10MB, đầy thì Docker tự xoay (rotate) sang file mới.
- `max-file: 3`: giữ tối đa 3 file cho mỗi container (file hiện tại + 2 file cũ), file cũ nhất bị xóa khi vượt quá.
- Cấu hình này áp dụng cho container **tạo mới sau khi restart Docker** — container đang chạy trước đó không tự nhận cấu hình mới.

```bash
# Cau hinh rieng cho tung container (ghi de cau hinh chung cua daemon)
docker run -d \
  --name payment-service \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  payment-service:v2.3
```

```bash
# Kiem tra dung luong log dang chiem cua tung container
sudo du -sh /var/lib/docker/containers/*/*-json.log 2>/dev/null | sort -rh | head -10
```

## Xem log bằng CLI thuần (khi chưa/không cần stack tập trung)

```bash
docker logs payment-service                 # xem toan bo log hien co
docker logs -f payment-service               # follow log real-time (giong tail -f)
docker logs --since 30m payment-service      # chi xem log 30 phut gan nhat
docker logs --tail 100 payment-service       # 100 dong cuoi cung
docker logs -t payment-service               # kem timestamp moi dong
```

## Docker Compose: dựng full stack Observability

```yaml
# docker-compose.yml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki
    restart: unless-stopped

  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command: ["run", "--server.http.listen-addr=0.0.0.0:12345", "/etc/alloy/config.alloy"]
    ports:
      - "12345:12345"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=MatKhauManh!23
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

volumes:
  prometheus-data:
  loki-data:
  grafana-data:
```

> [!warning] `privileged: true` của cAdvisor
> cAdvisor cần đọc thông tin cgroup/namespace ở mức sâu của kernel để đo tài nguyên chính xác cho từng container, nên thường cần chạy privileged hoặc mount đủ các đường dẫn `/sys`, `/var/run`, `/var/lib/docker` ở chế độ read-only như trên. Đây là điều cần cân nhắc về bảo mật — xem thêm nguyên tắc giảm quyền container ở [[../18-Security/README|Module 18]].

## Cấu hình Prometheus scrape cAdvisor

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```

## Cấu hình Grafana Alloy thu thập log container (thay thế Promtail)

```river
// config.alloy
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

loki.source.docker "default" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.docker.containers.targets
  labels     = {"env" = "production"}
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

- `discovery.docker`: tự động dò tìm mọi container đang chạy trên Docker daemon được trỏ tới, tương đương phần "service discovery" mà Promtail trước đây cấu hình bằng `docker_sd_configs`.
- `loki.source.docker`: đọc log của các container tìm được ở bước trên, gắn thêm label tĩnh (`env=production`) cộng với label tự động (`container`, `image`...), rồi chuyển tiếp (`forward_to`) sang component ghi dữ liệu.
- `loki.write`: component cuối cùng, thực sự gửi log qua HTTP tới Loki.

## PromQL — vài truy vấn cơ bản dùng thật trong vận hành

```promql
# CPU (giay/giay) tung container dang ton nhieu nhat, top 5
topk(5, rate(container_cpu_usage_seconds_total{name!=""}[5m]))

# Container nao dang gan cham RAM limit nhat (ty le %)
(container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100

# So luong container dang chay theo tung image
count by (image) (container_last_seen{image!=""})

# Container nao bi restart trong 15 phut qua (dua tren metric cAdvisor)
increase(container_start_time_seconds[15m]) > 0
```

## LogQL — vài truy vấn cơ bản dùng thật trong vận hành

```logql
# Toan bo log cua 1 container cu the
{container="payment-service"}

# Chi loc dong log co chua "ERROR"
{container="payment-service"} |= "ERROR"

# Dem so dong loi ERROR trong 1 gio qua, theo tung phut
sum by (container) (count_over_time({container="payment-service"} |= "ERROR" [1m]))

# Loc log dang JSON, lay truong "status_code" >= 500
{container="payment-service"} | json | status_code >= 500
```
