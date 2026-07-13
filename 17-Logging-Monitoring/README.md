---
title: "Module 17 - Logging & Monitoring"
tags: [docker, sysops-infra, module-17, logging, monitoring, loki, prometheus, grafana]
module: 17
part: "III - Operations"
difficulty: Advanced
status: draft
created: 2026-07-12
prerequisites: ["[[../16-Image-Registry/README|Module 16]]"]
next: "[[../18-Security/README|Module 18]]"
---

# Module 17 — Logging & Monitoring

> [!info] Vị trí trong giáo trình
> Part III — Operations · Module 17/21 của khóa 04 (System Operations & Infrastructure) · Tiếp nối Module 16 (Image Registry) — bạn đã biết lưu và phân phối image, giờ học cách **quan sát (observability)** những gì đang chạy từ image đó: có đang log lỗi không, có đang ăn hết tài nguyên không, có đang chết đi sống lại không.

> [!warning] Không thay thế Zabbix — bổ sung một tầng khác
> Bạn đã có tài liệu riêng về Zabbix ở khóa Linux Sysadmin, giám sát ở mức **host** (CPU/RAM/disk toàn máy, service systemd, thiết bị mạng qua SNMP...). Module này dạy bộ công cụ giám sát **container-native**: log và metric gắn liền với vòng đời từng container, sinh ra và biến mất theo container chứ không phải theo host — thứ Zabbix không được thiết kế để theo dõi ở độ chi tiết đó. Hai hệ thống bổ sung cho nhau trong một hạ tầng doanh nghiệp thực tế, không phải chọn một bỏ một.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Vì sao giám sát container khác giám sát host, mục tiêu học tập |
| [[02-Theory]] | Log driver, Loki, Promtail (và tình trạng EOL/Alloy), Prometheus, cAdvisor, Grafana |
| [[03-Architecture]] | Sơ đồ pipeline log, pipeline metric, internal working |
| [[04-Commands]] | Cấu hình log driver, docker-compose cho cả stack, PromQL/LogQL cơ bản |
| [[05-Labs]] | Lab log rotation, lab dựng full observability stack, lab dashboard + alert |
| [[06-Troubleshooting]] | Lỗi đầy đĩa do log, lỗi cAdvisor permission, lỗi target down, lỗi datasource |
| [[07-Interview]] | Câu hỏi phỏng vấn về logging/monitoring container |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 18 (Security) |

## Learning Objectives (tổng)

Sau khi hoàn thành module này, người học phải:

1. Giải thích được cơ chế log driver của Docker hoạt động thế nào, và vì sao driver mặc định `json-file` có thể làm đầy đĩa nếu không cấu hình rotation.
2. Phân biệt được mô hình **pull** (Prometheus tự kéo metrics theo chu kỳ) và **push** (agent tự đẩy log lên), giải thích lý do thiết kế của từng mô hình.
3. Tự dựng được một stack giám sát container-native hoàn chỉnh bằng Docker Compose: cAdvisor thu thập metrics container, Prometheus lưu trữ và truy vấn, Grafana Alloy thu thập log, Loki lưu trữ log, Grafana làm lớp hiển thị hợp nhất.
4. Viết được truy vấn PromQL và LogQL cơ bản để trả lời câu hỏi vận hành thực tế ("container nào đang ăn CPU cao nhất", "log lỗi nào xuất hiện nhiều nhất trong 1 giờ qua").
5. Xử lý được các sự cố thường gặp khi vận hành stack này: đĩa đầy vì log không rotate, cAdvisor không đọc được cgroup, Prometheus báo target down, Grafana không kết nối được datasource.
6. Biết chính xác ranh giới giữa giám sát container (module này) và giám sát host (Zabbix, đã học trước đó) để không nhầm lẫn phạm vi khi thiết kế hệ thống giám sát cho một hạ tầng thật.

## Self-Review

> [!note] Self-Review
> - **Technical Accuracy:** Đã WebSearch xác nhận (2026-07-12): **Promtail đã chuyển sang Long-Term Support từ 13/02/2025 và chính thức End-Of-Life ngày 02/03/2026** — tại thời điểm viết tài liệu này (07/2026), Promtail đã hết vòng đời. Module dạy khái niệm nền (log labeling, cách một agent tail log rồi gắn nhãn) qua Promtail vì đây vẫn là kiến thức áp dụng trực tiếp được, nhưng **thực hành (Lab, Commands) dùng Grafana Alloy** — agent hiện hành được Grafana khuyến nghị thay thế — và ghi chú rõ ràng tình trạng EOL ở mọi chỗ liên quan, không trình bày Promtail như công cụ hiện hành. cAdvisor (google/cadvisor) xác nhận vẫn đang được Google và cộng đồng bảo trì tích cực. Không bịa số liệu benchmark nào không kiểm chứng được.
> - **Completeness:** Đã bao phủ Docker log driver (json-file, local, journald, syslog), Loki + LogQL, Promtail/Alloy, Prometheus + PromQL, cAdvisor, Grafana, theo đúng phạm vi đề ra, có phân biệt rõ với Zabbix.
> - **ASCII Diagram:** Mọi sơ đồ trong [[03-Architecture]] đã được dựng bằng script Python (hàm `box()` đảm bảo `len()` border trên = border dưới = mọi dòng nội dung cho từng khung) và chạy script `verify_boxes.py` xác nhận "ALL BOXES OK" trước khi đưa vào file.

## Cấu trúc thư mục

```
17-Logging-Monitoring/
├── README.md
├── 01-Introduction.md
├── 02-Theory.md
├── 03-Architecture.md
├── 04-Commands.md
├── 05-Labs.md
├── 06-Troubleshooting.md
├── 07-Interview.md
├── 08-Summary.md
├── images/
├── labs/
└── scripts/
```
