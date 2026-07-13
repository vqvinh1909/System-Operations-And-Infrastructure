---
title: "03 - Architecture"
module: 17
tags: [docker, sysops-infra, module-17, logging, monitoring, architecture]
---

# 03 — Kiến trúc: Luồng Log và Luồng Metrics

> [!note] Cách đo sơ đồ trong file này
> Mọi khung (box) dưới đây được dựng bằng script Python nội bộ (hàm `box()` tính `inner_width = max(len(dòng)) + padding*2`, dùng đúng `inner_width` đó cho border trên, border dưới, và mọi dòng nội dung), sau đó chạy `verify_boxes.py` xác nhận `len(border trên) == len(border dưới) == len(mọi dòng nội dung)` cho từng khung — không đếm tay. Các dòng mũi tên/kết nối giữa hai khung (`|`, `v`, nhãn chú thích) không phải là "khung" nên không áp dụng quy tắc này.

## Sơ đồ 1: Docker Logging Driver — điểm khởi đầu của mọi log

```
+-------------------+
| CONTAINER PROCESS |
| (stdout / stderr) |
+-------------------+
         |
         v
+----------------------------+
| DOCKER LOGGING DRIVER      |
| json-file | local | syslog |
| journald | none | loki     |
+----------------------------+
              |
              v
+------------------------------------------+
| DIEM DEN (DESTINATION)                   |
| File tren host  ->  Log server  ->  Loki |
+------------------------------------------+
```

Đây là tầng thấp nhất, xảy ra ngay trong Docker daemon, trước khi bất kỳ công cụ giám sát bên ngoài nào tham gia. Chọn sai driver hoặc không cấu hình rotation ở tầng này sẽ gây sự cố (đầy đĩa) mà không công cụ nào ở các tầng phía trên phát hiện kịp thời — vì Prometheus/Loki chỉ giám sát những gì được expose ra, không tự giám sát chính hạ tầng logging của Docker.

## Sơ đồ 2: Pipeline Metrics — Container → cAdvisor → Prometheus → Grafana

```
+-------------------------+
| CONTAINER 1..N          |
| (app chay trong Docker) |
+-------------------------+
  |
  v
+----------------------------------------+
| cAdvisor                               |
| expose :8080/metrics                   |
| (CPU, RAM, I/O, network moi container) |
+----------------------------------------+
  | Prometheus pull moi 15s
  v
+-----------------------------------+
| PROMETHEUS                        |
| TSDB + PromQL + Alertmanager rule |
+-----------------------------------+
  |
  v
+---------------------------------+
| GRAFANA                         |
| dashboard truc quan hoa metrics |
+---------------------------------+
```

Lưu ý mũi tên giữa cAdvisor và Prometheus đi **ngược hướng thực tế của kết nối mạng** so với cách đọc thông thường: Prometheus mới là bên **chủ động** gửi HTTP GET tới cAdvisor theo chu kỳ (mô hình pull đã học ở [[02-Theory]]), không phải cAdvisor tự đẩy dữ liệu đi. Sơ đồ vẽ theo hướng "dữ liệu chảy về đâu" để dễ hình dung luồng thông tin, không phải hướng khởi tạo kết nối TCP.

## Sơ đồ 3: Pipeline Logs — Container → Grafana Alloy → Loki → Grafana

```
+-------------------+
| CONTAINER 1..N    |
| (stdout / stderr) |
+-------------------+
  |
  v
+-----------------------------------+
| GRAFANA ALLOY                     |
| (thay the Promtail - EOL 03/2026) |
| tail log file, gan label          |
+-----------------------------------+
  | push log co label
  v
+-----------------------------------+
| LOKI                              |
| luu log nen (chunk),              |
| chi index label - khong full-text |
+-----------------------------------+
  |
  v
+--------------------------------------------+
| GRAFANA                                    |
| Explore: query LogQL, lien ket voi metrics |
+--------------------------------------------+
```

Khác với pipeline Metrics, pipeline Logs đi theo mô hình **push**: Grafana Alloy là bên chủ động đọc file log (thường là chính file `json-file` do log driver tạo ra tại `/var/lib/docker/containers/<id>/*.log` trên host, hoặc đọc trực tiếp qua Docker API), gắn thêm label (`container_name`, `job`, host...), rồi tự gửi dữ liệu tới Loki. Đây là lý do Alloy cần chạy trên **mọi host** có container cần thu thập log — nó là một agent tại chỗ, không phải dịch vụ trung tâm kéo dữ liệu về như Prometheus.

## Internal Working: vì sao hai pipeline tách biệt lại hội tụ đúng một điểm trong Grafana

1. Cả Prometheus và Loki đều được khai báo là **datasource** riêng biệt trong Grafana — về mặt kỹ thuật chúng không biết tới sự tồn tại của nhau.
2. Grafana là nơi duy nhất "biết" cách liên kết hai nguồn dữ liệu này lại: một dashboard metric (nguồn Prometheus) hiển thị biểu đồ CPU container tăng vọt lúc 2h07, người vận hành click vào điểm dữ liệu đó, Grafana tạo sẵn một truy vấn LogQL tương ứng (lọc theo `container_name` trùng khớp và khung thời gian trùng khớp) rồi mở tab Explore trỏ tới datasource Loki.
3. Sự liên kết này **không tự động xảy ra** nếu bạn không đặt cùng một label nhất quán ở cả hai phía (ví dụ cAdvisor và Alloy phải cùng dùng tên container thật làm giá trị label, không được một bên dùng container ID, một bên dùng container name) — đây là lỗi cấu hình phổ biến khiến "liên kết metric-log" trên lý thuyết không hoạt động trên thực tế, xem thêm [[06-Troubleshooting]].

> [!info] So với Zabbix
> Zabbix có một agent duy nhất thu thập cả metric lẫn log ở mức host, gửi về một server Zabbix duy nhất — kiến trúc tập trung, đơn giản hơn nhưng không phân biệt được từng container. Stack ở module này cố tình tách hai pipeline độc lập (Prometheus riêng, Loki riêng) vì khối lượng dữ liệu container-level lớn hơn nhiều bậc so với host-level, cần hai hệ thống lưu trữ chuyên biệt được tối ưu riêng cho từng loại dữ liệu (time series số liệu vs. luồng văn bản log).
