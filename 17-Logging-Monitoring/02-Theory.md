---
title: "02 - Theory"
module: 17
tags: [docker, sysops-infra, module-17, logging, monitoring, loki, prometheus]
---

# 02 — Lý thuyết: Log Driver, Loki, Prometheus, cAdvisor, Grafana

## 1. Docker Logging Driver — nền tảng của mọi thứ phía sau

Mỗi khi một tiến trình bên trong container ghi ra `stdout`/`stderr`, Docker daemon không để dữ liệu đó "bay mất" — nó chặn (capture) luồng output này và giao cho một **logging driver** xử lý. Driver nào được dùng do cấu hình `--log-driver` (per-container) hoặc `log-driver` trong `/etc/docker/daemon.json` (mặc định cho toàn bộ host) quyết định.

Các driver phổ biến:

| Driver | Nơi lưu | Đặc điểm |
|---|---|---|
| `json-file` | File JSON trên host, 1 dòng/log entry | **Mặc định** của Docker. Không giới hạn dung lượng nếu không cấu hình `max-size`/`max-file`. |
| `local` | Định dạng nhị phân nén trên host | Hiệu năng tốt hơn `json-file`, có rotation mặc định, nhưng không tương thích trực tiếp với `docker logs` cũ và một số tool bên ngoài. |
| `journald` | systemd-journald | Hợp nhất log container vào cùng hệ thống journal của host — hữu ích nếu bạn đã dùng `journalctl` để tra cứu log hệ thống. |
| `syslog` | Syslog server (rsyslog, syslog-ng...) | Phù hợp hạ tầng đã có sẵn syslog tập trung. |
| `none` | Không lưu gì cả | Dùng khi ứng dụng tự ghi log ra nơi khác (ví dụ gọi thẳng API của một dịch vụ log tập trung), tránh lưu trùng lặp hai lần. |

> [!warning] Vì sao `json-file` mặc định là một rủi ro vận hành thường bị bỏ qua
> Nếu không cấu hình `max-size` và `max-file`, một container log liên tục (ví dụ do vòng lặp lỗi ghi log mỗi request) có thể lấp đầy toàn bộ dung lượng đĩa `/var/lib/docker` trong vài giờ — kéo theo daemon Docker không thể ghi thêm dữ liệu, ảnh hưởng **mọi** container khác trên cùng host, không chỉ container đang lỗi. Đây là một trong những nguyên nhân sự cố production phổ biến nhất liên quan tới Docker, và cách khắc phục (cấu hình rotation) rất đơn giản nhưng thường bị bỏ qua lúc mới dựng hệ thống. Xem cấu hình cụ thể ở [[04-Commands]].

Dù chọn driver nào, log driver chỉ giải quyết bài toán "log không mất khi container còn tồn tại". Nó **không** giải quyết bài toán tra cứu tập trung xuyên nhiều container/nhiều host — đó là lý do cần thêm một tầng thu thập log (log aggregation), chính là Loki.

## 2. Loki — kho lưu trữ log "index theo label, không full-text"

Loki là hệ thống lưu trữ và truy vấn log do Grafana Labs phát triển, thiết kế theo triết lý khác hẳn Elasticsearch (ELK stack quen thuộc): **Loki không index toàn bộ nội dung log**, nó chỉ index một tập nhỏ **label** (nhãn) gắn với mỗi luồng log — ví dụ `container_name`, `job`, `host`. Nội dung log thật được nén và lưu thành các "chunk", chỉ được quét (grep) khi có truy vấn khớp label trước.

Hệ quả trực tiếp:
- Loki rẻ hơn nhiều lần so với Elasticsearch ở cùng khối lượng log, vì không tốn chi phí index full-text.
- Truy vấn Loki **luôn phải bắt đầu bằng một bộ lọc label** (ví dụ `{container_name="payment-service"}`), sau đó mới lọc tiếp theo nội dung bằng biểu thức LogQL (ví dụ `|= "ERROR"`). Nếu bạn cố truy vấn không kèm label nào, Loki sẽ từ chối hoặc quét toàn bộ dữ liệu cực chậm.
- Loki phù hợp nhất khi ghép cùng Grafana — vì bản thân triết lý thiết kế của nó giả định người dùng luôn "nhảy" từ một dashboard metric (đã biết thời điểm, đã biết container nào bất thường) sang xem log tương ứng, chứ không phải tìm kiếm log tự do như công cụ search engine.

## 3. Agent thu thập log: Promtail (đã EOL) và Grafana Alloy (hiện hành)

Loki tự nó không đi đọc log — cần một **agent** chạy trên mỗi host, tail (đọc liên tục) các file log hoặc log driver output, gắn label, rồi đẩy (push) vào Loki.

- **Promtail** từng là agent chính thức và phổ biến nhất cho Loki trong nhiều năm. Tuy nhiên, **Promtail đã chuyển sang Long-Term Support (LTS) từ 13/02/2025 và chính thức End-Of-Life (EOL) ngày 02/03/2026** — nghĩa là tại thời điểm bạn học module này, Promtail không còn được phát triển tính năng mới, chỉ còn vá lỗi bảo mật nghiêm trọng trong giai đoạn LTS, và đã hết hạn ngay cả giai đoạn đó. Nhiều tài liệu/blog cũ trên internet vẫn hướng dẫn dùng Promtail — cần nhận biết đây là kiến thức lỗi thời khi tra cứu thêm.
- **Grafana Alloy** là agent thế hệ mới, được xây trên nền OpenTelemetry Collector, là công cụ Grafana Labs khuyến nghị thay thế Promtail cho mọi triển khai mới. Alloy không chỉ thu thập log — nó là một collector đa năng (log, metric, trace, profiling) dùng chung một binary, cấu hình bằng ngôn ngữ riêng gọi là Alloy configuration syntax (dựa trên HCL). Grafana cung cấp sẵn công cụ chuyển đổi cấu hình Promtail cũ sang Alloy tự động.

Module này dạy khái niệm gắn label qua ví dụ Promtail (vì cú pháp đơn giản, dễ hình dung), nhưng **toàn bộ phần thực hành (Commands, Labs) dùng Grafana Alloy** để bạn thực hành đúng công cụ đang được duy trì.

## 4. Prometheus — mô hình pull, TSDB, PromQL

Prometheus là hệ thống giám sát metric theo mô hình **pull**: thay vì chờ ứng dụng tự đẩy số liệu tới, Prometheus **chủ động gửi HTTP request tới một endpoint `/metrics`** của từng target theo chu kỳ cố định (mặc định 15 giây, gọi là "scrape interval"), rồi lưu kết quả vào Time Series Database (TSDB) nội bộ.

Vì sao pull thay vì push?
- **Đơn giản hóa việc phát hiện target sống/chết**: nếu Prometheus không scrape được một target trong khoảng thời gian dự kiến, nó tự động biết target đó đang down — không cần cơ chế heartbeat riêng.
- **Kiểm soát tải tập trung**: Prometheus tự quyết định tần suất scrape, tránh trường hợp hàng trăm ứng dụng cùng lúc "dội" dữ liệu về server giám sát gây quá tải.
- Đánh đổi: ứng dụng/target phải tự expose một endpoint HTTP đúng định dạng Prometheus — đây là lý do cần các "exporter" (như cAdvisor) đứng giữa để dịch dữ liệu nội bộ (cgroup, hệ thống...) sang định dạng Prometheus hiểu được.

Dữ liệu trong Prometheus là các **time series** — mỗi chuỗi số liệu gắn với một tên metric và tập hợp cặp key-value gọi là **label** (ví dụ `container_cpu_usage_seconds_total{name="payment-service"}`). Bốn loại metric cơ bản: `Counter` (chỉ tăng, ví dụ tổng số request), `Gauge` (lên xuống tự do, ví dụ RAM đang dùng), `Histogram` và `Summary` (phân phối giá trị, ví dụ độ trễ request).

**PromQL** là ngôn ngữ truy vấn của Prometheus, dùng để lọc, tổng hợp, tính toán trên các time series này — ví dụ tính tốc độ tăng của một counter theo thời gian bằng hàm `rate()`.

## 5. cAdvisor — cầu nối giữa cgroup của container và Prometheus

**cAdvisor** (Container Advisor) là dự án do Google phát triển và duy trì, chạy như một container trên chính host cần giám sát. Nó đọc trực tiếp thông tin từ **cgroup** (control group — cơ chế kernel Linux giới hạn/đo tài nguyên mà mỗi container sử dụng, đã học khái niệm ở Part II khi tìm hiểu container runtime) của từng container, rồi expose toàn bộ dữ liệu đó dưới dạng endpoint `/metrics` đúng chuẩn Prometheus tại cổng `:8080`.

Nhờ cAdvisor, bạn có được các chỉ số ở mức **từng container riêng lẻ** mà một công cụ giám sát host-level như Zabbix không cung cấp: CPU usage theo container, RAM usage/limit theo container, network I/O theo container, disk I/O theo container — tất cả đã gắn sẵn label `name`/`id` để Prometheus phân biệt được container nào với container nào.

## 6. Grafana — lớp hiển thị hợp nhất

Grafana không tự lưu trữ dữ liệu — nó là lớp **truy vấn và trực quan hóa** kết nối tới nhiều nguồn dữ liệu (datasource) cùng lúc: Prometheus cho metric, Loki cho log, và nhiều loại khác (không thuộc phạm vi module này). Điểm mạnh cốt lõi khi ghép Prometheus + Loki + Grafana: từ một điểm bất thường trên biểu đồ metric (ví dụ CPU tăng vọt lúc 2h07), bạn có thể click thẳng sang xem log của đúng container đó trong đúng khung giờ đó (tính năng "Explore" liên kết metric-log), rút ngắn đáng kể thời gian điều tra sự cố so với việc mở hai công cụ tách biệt.

Grafana còn hỗ trợ **Alerting**: định nghĩa rule dựa trên PromQL/LogQL, tự động gửi cảnh báo (qua email, Slack, webhook...) khi điều kiện vi phạm — chuyển vai trò từ "phải tự mở dashboard để nhìn" sang "hệ thống tự báo khi có vấn đề".

## 7. Tổng hợp: hai pipeline độc lập, hội tụ tại Grafana

| | Pipeline Metric | Pipeline Log |
|---|---|---|
| Nguồn dữ liệu | cgroup của container | stdout/stderr của container |
| Công cụ thu thập | cAdvisor (expose `/metrics`) | Grafana Alloy (tail + gắn label) |
| Mô hình | Pull (Prometheus tự kéo) | Push (Alloy tự đẩy) |
| Nơi lưu trữ | Prometheus (TSDB) | Loki (chunk nén, index label) |
| Ngôn ngữ truy vấn | PromQL | LogQL |
| Hiển thị | Grafana | Grafana |

Chi tiết sơ đồ luồng dữ liệu của cả hai pipeline xem tại [[03-Architecture]].
