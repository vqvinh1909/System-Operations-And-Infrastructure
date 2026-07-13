---
title: "08 - Summary"
module: 19
tags: [docker, sysops-infra, module-19, troubleshooting, summary]
---

# 08 — Tóm tắt Module 19

## Những gì bạn đã học

- Một **mô hình debug phân tầng** (Application → Container → Network → Storage → Host/Kernel) để khoanh vùng bất kỳ sự cố Docker nào một cách có hệ thống, thay vì thử ngẫu nhiên.
- **Container Debugging**: phân biệt `Exited` và `Restarting`, đọc đúng exit code (đặc biệt 137 = OOM), dùng `docker inspect`/`docker events`/`docker exec` để thu thập bằng chứng thay vì phỏng đoán.
- **Network Debugging**: quy trình "từ trong ra ngoài" (bên trong container → cùng network → port mapping → firewall/external), lỗi phổ biến nhất là dùng sai hostname (`localhost` thay vì tên service).
- **Storage Debugging**: phân biệt vấn đề permission (UID lệch giữa host/container), dung lượng (image, container, volume, và build cache — nguồn hay bị bỏ sót), và integrity của storage driver.
- **Performance Analysis**: luôn đối chiếu số liệu riêng của container (`docker stats`, cgroup) với tài nguyên tổng thể của host (`vmstat`, `top`, `iostat`) — phân biệt "giới hạn cấu hình sai" và "host thật sự quá tải" là kỹ năng quan trọng nhất của tầng này.
- Một **checklist chẩn đoán nhanh** 8 bước áp dụng được ngay khi chưa biết bắt đầu từ đâu, tổng hợp toàn bộ 4 phần ở [[06-Troubleshooting]].

## Bảng tổng hợp: 4 tầng, câu hỏi cốt lõi, công cụ chính

| Tầng | Câu hỏi cốt lõi | Công cụ chính |
|---|---|---|
| Container | Container có chạy đúng vòng đời kỳ vọng không? | `docker inspect`, `docker logs`, `docker events` |
| Network | Hai điểm có "thấy" nhau đúng theo cấu hình không? | `docker network inspect`, DNS nội bộ, iptables |
| Storage | Quyền và dung lượng có đúng như kỳ vọng không? | `docker system df`, UID đối chiếu, `df -h` |
| Performance | Bottleneck nằm ở container hay ở host tổng thể? | `docker stats`, cgroup trực tiếp, `vmstat`/`iostat` |

## Cầu nối sang Module 20

Bạn đã đi qua toàn bộ hành trình từ Ansible (Part I) tới Docker (Part II) tới vận hành hạ tầng doanh nghiệp (Part III: Registry, Logging & Monitoring, Security, và giờ là Troubleshooting). [[../20-Enterprise-Project/README|Module 20 — Enterprise Project]] là bài kiểm tra tổng hợp cuối cùng: bạn sẽ tự dựng một hạ tầng 5 VM hoàn chỉnh (Control Node, HAProxy, hai Web server, một Database server), triển khai bằng Ansible, deploy ứng dụng bằng Docker Compose, gắn giám sát, thiết lập backup — và tự chẩn đoán 20 tình huống sự cố thực tế **không có đáp án dẫn sẵn từng bước**, đúng như những gì bạn sẽ gặp trong công việc thật. Toàn bộ kỹ năng chẩn đoán học ở module này chính là công cụ bạn sẽ dùng để hoàn thành phần đó.
