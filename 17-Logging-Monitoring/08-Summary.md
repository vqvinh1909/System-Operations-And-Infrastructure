---
title: "08 - Summary"
module: 17
tags: [docker, sysops-infra, module-17, logging, monitoring, summary]
---

# 08 — Tóm tắt Module 17

## Những gì bạn đã học

- **Docker Logging Driver** là tầng nền tảng đầu tiên — `json-file` mặc định không giới hạn dung lượng nếu không cấu hình `max-size`/`max-file`, và log driver không giải quyết bài toán tra cứu tập trung xuyên nhiều container/host.
- **Loki** lưu trữ log theo triết lý "index label, không full-text" — rẻ hơn Elasticsearch nhưng bắt buộc mọi truy vấn LogQL phải bắt đầu bằng bộ lọc label.
- **Grafana Alloy** là agent hiện hành để thu thập log container, thay thế **Promtail** đã chính thức End-Of-Life từ 02/03/2026.
- **Prometheus** thu thập metric theo mô hình pull, lưu trong TSDB nội bộ, truy vấn bằng PromQL.
- **cAdvisor** là cầu nối bắt buộc giữa cgroup của kernel và Prometheus — không có nó, Prometheus không có cách nào biết được tài nguyên từng container đang dùng bao nhiêu.
- **Grafana** là lớp hiển thị hợp nhất, liên kết dashboard metric (Prometheus) với log tương ứng (Loki) tại đúng thời điểm và đúng container, rút ngắn thời gian điều tra sự cố.
- Stack này **bổ sung**, không thay thế Zabbix — Zabbix vẫn là công cụ đúng cho giám sát host-level, còn stack module này chuyên cho giám sát container-level.

## Bảng đối chiếu nhanh

| Tiêu chí | Zabbix (đã học) | Stack Module 17 |
|---|---|---|
| Phạm vi giám sát | Host, service hệ thống, SNMP | Từng container riêng lẻ |
| Mô hình thu thập | Agent + server tập trung | Pull (metric) + Push (log), 2 pipeline tách biệt |
| Nơi lưu metric | Zabbix Server DB | Prometheus TSDB |
| Nơi lưu log | Thường tích hợp ngoài hoặc syslog | Loki |
| Ngôn ngữ truy vấn | Zabbix trigger expression | PromQL / LogQL |
| Phù hợp nhất | Giám sát hạ tầng vật lý/VM | Giám sát ứng dụng chạy trong container |

## Cầu nối sang Module 18

Bạn giờ đã "nhìn thấy" được container đang chạy — biết nó log gì, ăn bao nhiêu tài nguyên, khi nào bất thường. Nhưng quan sát được không có nghĩa là container đó **an toàn**. Một container chạy bằng quyền root không giới hạn, không seccomp, không AppArmor, image chưa từng được scan lỗ hổng — vẫn có thể là điểm xâm nhập cho kẻ tấn công dù dashboard Grafana vẫn "xanh" bình thường. Đó chính là nội dung [[../18-Security/README|Module 18 — Security]], nơi bạn học cách thu hẹp bề mặt tấn công (attack surface) của container bằng rootless mode, capability, seccomp, AppArmor và image scanning.
