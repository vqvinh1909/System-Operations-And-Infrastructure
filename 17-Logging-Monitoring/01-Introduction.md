---
title: "01 - Introduction"
module: 17
tags: [docker, sysops-infra, module-17, logging, monitoring]
---

# 01 — Giới thiệu Module: Logging & Monitoring

## Tình huống mở đầu

Container `payment-service` restart lúc 2h sáng. Không ai chứng kiến trực tiếp. Đến 8h sáng, khách hàng báo cáo giao dịch bị lỗi trong khoảng 2h05–2h12. Bạn chạy `docker logs payment-service` để xem log — nhưng vì `docker compose up` đã deploy một bản mới trong đêm (container cũ bị xóa, container mới được tạo ra với ID khác), toàn bộ log của container cũ **biến mất hoàn toàn cùng với container đó**. Log driver mặc định `json-file` lưu log gắn liền với container ID cụ thể trên đĩa host — không có khái niệm "lịch sử log xuyên suốt nhiều lần deploy" nếu không có gì thu thập nó ra ngoài.

Đây chính là vấn đề cốt lõi mà module này giải quyết: log và metric của container có vòng đời gắn liền với container, trong khi bạn cần khả năng tra cứu **xuyên suốt thời gian, xuyên suốt nhiều container, nhiều host**, kể cả sau khi container gốc đã không còn tồn tại.

## Vì sao module này quan trọng với vai trò Sysadmin/DevOps

Bạn đã có Zabbix để biết "server này CPU 95%, RAM sắp đầy, disk còn 5%". Nhưng Zabbix trả lời câu hỏi ở mức **host**, không trả lời được: "Trong 20 container đang chạy trên host này, container nào đang ăn hết CPU?", "Container `payment-service` in ra log lỗi gì lúc 2h07 sáng qua, ngay trước khi nó bị OOM-killed?". Đây chính là khoảng trống mà bộ công cụ container-native (Docker log driver, Loki, Grafana Alloy, Prometheus, cAdvisor, Grafana) lấp vào.

Trong công việc thực tế, khi một sự cố production xảy ra với hệ thống chạy Docker, quy trình điều tra luôn đi qua ba câu hỏi theo thứ tự:

1. **Cái gì đã xảy ra** (log) — container nói gì trước khi lỗi.
2. **Khi nào và mức độ nào** (metric) — CPU/RAM/network tăng đột biến lúc nào, bao nhiêu.
3. **Có liên quan đến deploy/thay đổi nào không** — đối chiếu thời điểm log/metric bất thường với lịch sử deploy (image nào được push/pull lúc nào, đã học ở Module 16).

Không có logging/monitoring tập trung, cả ba câu hỏi này chỉ trả lời được nếu bạn may mắn còn giữ được container gốc — điều gần như không bao giờ đúng trong môi trường production thật, nơi container bị xóa và tạo lại liên tục theo mỗi lần deploy hoặc mỗi lần healthcheck fail.

## Mục tiêu học tập

Sau module này, bạn sẽ:

1. Hiểu cơ chế log driver của Docker, biết vì sao log driver mặc định không đủ cho production quy mô lớn nếu không cấu hình đúng.
2. Dựng được pipeline log tập trung: agent thu thập log (Grafana Alloy — công cụ hiện hành, thay thế Promtail đã hết vòng đời từ 03/2026) → Loki lưu trữ → Grafana truy vấn/hiển thị.
3. Dựng được pipeline metric tập trung: cAdvisor xuất metric container → Prometheus scrape và lưu trữ → Grafana hiển thị dashboard.
4. Viết được truy vấn LogQL/PromQL cơ bản để tự điều tra sự cố thay vì chỉ nhìn dashboard có sẵn.
5. Biết chính xác ranh giới giữa giám sát container (module này) và giám sát host (Zabbix, đã học trước đó) để không nhầm lẫn phạm vi khi thiết kế hệ thống giám sát cho một hạ tầng thật.

## Kiến thức nền cần có trước

- Đã hoàn thành Module 16 (Image Registry).
- Hiểu Docker Compose cơ bản (Part II) — stack ở module này sẽ được dựng hoàn toàn bằng Docker Compose.
- Đã có kiến thức Zabbix từ khóa Linux Sysadmin trước đó — module này sẽ liên tục đối chiếu để bạn thấy rõ sự khác biệt, không học lại từ đầu khái niệm giám sát nói chung.
