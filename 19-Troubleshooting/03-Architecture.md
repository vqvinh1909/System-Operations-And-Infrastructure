---
title: "03 - Architecture"
module: 19
tags: [docker, sysops-infra, module-19, troubleshooting, architecture]
---

# 03 — Kiến trúc: Mô hình Phân Tầng và Luồng Chẩn Đoán

> [!note] Cách đo sơ đồ trong file này
> Mọi khung (box) dưới đây được dựng bằng script Python nội bộ (hàm `box()` tính `inner_width = max(len(dòng)) + padding*2`, dùng đúng `inner_width` đó cho border trên, border dưới, và mọi dòng nội dung), sau đó chạy `verify_boxes.py` xác nhận `len()` khớp nhau cho từng khung — không đếm tay.

## Sơ đồ 1: Mô hình debug 5 tầng — đi từ ngoài vào trong

```
+-----------------------------------------+
| LOP 1 - APPLICATION                     |
| log loi trong app, exit code bat thuong |
+-----------------------------------------+
  |
  v
+------------------------------------------+
| LOP 2 - CONTAINER                        |
| docker inspect, docker logs, docker exec |
+------------------------------------------+
  |
  v
+------------------------------------------------+
| LOP 3 - NETWORK                                |
| docker network inspect, DNS, iptables/nftables |
+------------------------------------------------+
  |
  v
+--------------------------------------------------+
| LOP 4 - STORAGE                                  |
| volume, bind mount, driver, dung dia (disk full) |
+--------------------------------------------------+
  |
  v
+---------------------------------------------------+
| LOP 5 - HOST / KERNEL                             |
| cgroup, namespace, tai nguyen vat ly (CPU/RAM/IO) |
+---------------------------------------------------+
```

Đây là sơ đồ tổng quát nhất của toàn module — mọi kịch bản chi tiết ở [[06-Troubleshooting]] đều là một lát cắt cụ thể xuyên qua các tầng này. Lưu ý: thứ tự đi từ trên xuống không có nghĩa lỗi luôn nằm ở tầng dưới cùng — nó có nghĩa **bạn kiểm tra theo thứ tự đó để loại trừ có hệ thống**, và dừng lại ngay khi tìm thấy bằng chứng rõ ràng ở tầng nào.

## Sơ đồ 2: Luồng chẩn đoán Network cụ thể

```
+-----------------------------------------------------+
| Trieu chung: container A khong goi duoc container B |
+-----------------------------------------------------+
  |
  v
+---------------------------------------------+
| Buoc 1: docker exec A ping B                |
| kiem tra DNS noi bo cua Docker (127.0.0.11) |
+---------------------------------------------+
  |
  v
+------------------------------------------+
| Buoc 2: docker network inspect <network> |
| kiem tra A, B co chung 1 network khong   |
+------------------------------------------+
  |
  v
+-------------------------------------+
| Buoc 3: kiem tra iptables tren host |
| iptables -t nat -L DOCKER -n -v     |
+-------------------------------------+
  |
  v
+---------------------------------------------+
| Buoc 4: kiem tra bridge + routing tren host |
| ip addr, ip route, brctl show               |
+---------------------------------------------+
```

Đây là ví dụ áp dụng cụ thể của "Lớp 3 - Network" trong Sơ đồ 1, minh họa đúng nguyên tắc "từ trong ra ngoài" đã học ở [[02-Theory]]: bắt đầu kiểm tra từ chính bên trong container nghi vấn, chỉ mở rộng ra host khi các bước gần hơn đã loại trừ.

## Sơ đồ 3: Luồng chẩn đoán Performance cụ thể

```
+-------------------------------------------------+
| Trieu chung: container chay cham / bi OOMKilled |
+-------------------------------------------------+
  |
  v
+---------------------------------------+
| Buoc 1: docker stats                  |
| xem nhanh CPU %, MEM usage/limit, I/O |
+---------------------------------------+
  |
  v
+------------------------------------------+
| Buoc 2: doc cgroup truc tiep             |
| /sys/fs/cgroup/.../memory.stat, cpu.stat |
+------------------------------------------+
  |
  v
+-----------------------------------+
| Buoc 3: doi chieu tai nguyen host |
| vmstat 1, iostat -x 1, top        |
+-----------------------------------+
  |
  v
+---------------------------------------------------------------+
| Buoc 4: ket luan - do app, do limit sai, hay do host qua tai? |
+---------------------------------------------------------------+
```

`docker stats` (Bước 1) là điểm khởi đầu nhanh nhưng **chỉ là số liệu đã được tổng hợp sẵn** — khi số liệu đó cho thấy dấu hiệu bất thường (CPU sát 100%, MEM sát limit), Bước 2 xác nhận lại bằng cách đọc trực tiếp file cgroup (nguồn dữ liệu gốc mà `docker stats` cũng lấy từ đó) để loại trừ khả năng chính công cụ đo lường báo sai. Bước 3 là bước hay bị bỏ qua nhất trong thực tế — nhiều người dừng lại ở Bước 1-2 và kết luận vội "do container cấu hình sai", trong khi nếu không đối chiếu với tài nguyên tổng của host, kết luận đó có thể sai hoàn toàn (xem kịch bản cụ thể ở [[06-Troubleshooting]]).

## Internal Working: vì sao `docker stats` và cgroup đôi khi cho số liệu lệch nhau

`docker stats` không tự đo tài nguyên — nó đọc dữ liệu **gốc** từ đúng những file cgroup mà Bước 2 ở Sơ đồ 3 hướng dẫn đọc trực tiếp, rồi tính toán/định dạng lại cho dễ đọc (ví dụ tính % CPU dựa trên khoảng thời gian lấy mẫu). Vì có một bước tính toán trung gian, trong một số trường hợp hiếm (host tải rất cao, độ trễ lấy mẫu lớn), số liệu `docker stats` hiển thị có thể trễ hoặc làm tròn khác với việc đọc trực tiếp file cgroup tại đúng thời điểm nghi vấn — đây là lý do khi cần độ chính xác cao để chẩn đoán sự cố nghiêm trọng, luôn nên đối chiếu cả hai nguồn thay vì chỉ tin tưởng một công cụ tổng hợp sẵn.
