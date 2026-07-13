---
title: "03 - Architecture"
module: 20
tags: [docker, ansible, sysops-infra, module-20, capstone, architecture]
---

# 03 — Kiến trúc tổng thể: Hạ tầng 5 VM

> [!note] Cách đo sơ đồ trong file này
> Sơ đồ dưới đây được dựng bằng script Python nội bộ (hàm `box()` tính `inner_width = max(len(dòng)) + padding*2`, dùng đúng `inner_width` đó cho border trên, border dưới, và mọi dòng nội dung của từng khung), sau đó chạy `verify_boxes.py` xác nhận "ALL BOXES OK" trước khi đưa vào file. Đây là **sơ đồ quan trọng nhất toàn khóa** — mọi bước triển khai ở [[05-Labs]] và mọi tình huống sự cố ở [[06-Troubleshooting]] đều xoay quanh đúng 5 khối và đúng luồng kết nối trong sơ đồ này.

## Sơ đồ 1: Kiến trúc tổng thể 5 VM

```
+--------------------------------------+
| CONTROL NODE (VM1) - 192.168.10.10   |
| Ansible: inventory, playbooks, roles |
| quan ly ca 4 node duoi qua SSH key   |
+--------------------------------------+
  | (buoc 1: ssh + ansible-playbook site.yml)
  v
+-----------------------------------------------------+
| HAPROXY (VM2) - 192.168.10.20                       |
| Load Balancer / Reverse Proxy (Keepalived tuy chon) |
| listen :80, :443 -> backend web_servers             |
+-----------------------------------------------------+
  | (buoc 2: HAProxy phan phoi request)
  v
           |                                  |           
           v                                  v           
+---------------------+            +---------------------+
| WEB01 (VM3)         |            | WEB02 (VM4)         |
| 192.168.10.31       |            | 192.168.10.32       |
| docker: nginx + app |            | docker: nginx + app |
+---------------------+            +---------------------+
           \                                  /           
                             v  (buoc 3: ca 2 web ghi/doc chung 1 database)
           +----------------------------------+
           | DATABASE (VM5) - 192.168.10.40   |
           | docker: mysql + redis + rabbitmq |
           | volume: /data/mysql (persistent) |
           +----------------------------------+
```

Đọc sơ đồ theo hai luồng khác nhau, không được nhầm lẫn:

1. **Luồng triển khai (control plane)** — Control Node dùng Ansible để cấu hình **cả 4 node còn lại**, không chỉ HAProxy. Mũi tên "bước 1" trong sơ đồ chỉ minh họa điểm khởi đầu của luồng SSH, không có nghĩa Control Node chỉ nói chuyện với HAProxy — chi tiết đầy đủ xem Sơ đồ 2.
2. **Luồng traffic thật (data plane)** — người dùng cuối gọi vào HAProxy, HAProxy phân phối sang Web01/Web02, cả hai cùng đọc/ghi chung một Database — đây là luồng chạy liên tục 24/7 sau khi đã triển khai xong, hoàn toàn độc lập với Control Node (Control Node **không** nằm trên đường đi của traffic thật, đúng nguyên tắc tách biệt control plane và data plane).

## Sơ đồ 2: Luồng Ansible từ Control Node

```
+--------------------------------------+
| CONTROL NODE                         |
| ansible.cfg + inventory.ini + roles/ |
+--------------------------------------+
  | ansible-playbook -i inventory site.yml
  v
+---------------------------------------+
| SSH (public key, khong dung password) |
+---------------------------------------+
  | ket noi song song toi 4 managed node
  v
+----------------------------------------------------+
| HAPROXY  |  WEB01  |  WEB02  |  DATABASE           |
| (moi node co role rieng: haproxy, docker, app, db) |
+----------------------------------------------------+
```

Ansible kết nối **song song** (không tuần tự) tới cả 4 managed node trong cùng một lần chạy `ansible-playbook`, mỗi node áp dụng đúng role tương ứng với vai trò của nó (node HAProxy chỉ nhận role `haproxy`, node Database chỉ nhận role `database`...) — đây chính là mô hình Roles đã học ở Module 06, áp dụng lại nguyên vẹn ở quy mô một dự án thật thay vì bài lab đơn lẻ.

## Bảng địa chỉ IP tham khảo (dùng xuyên suốt các file khác của module)

| VM | Vai trò | IP (ví dụ, tùy chỉnh theo lab thật của bạn) |
|---|---|---|
| VM1 | Control Node | 192.168.10.10 |
| VM2 | HAProxy | 192.168.10.20 |
| VM3 | Web01 | 192.168.10.31 |
| VM4 | Web02 | 192.168.10.32 |
| VM5 | Database (MySQL + Redis + RabbitMQ) | 192.168.10.40 |

> [!info] Vì sao Database "gộp" cả MySQL, Redis, RabbitMQ trên cùng một VM
> Trong doanh nghiệp thật với ngân sách lớn hơn, ba dịch vụ này thường được tách thành 3 VM riêng để scale và cô lập rủi ro độc lập. Ở quy mô đồ án học tập với 5 VM, gộp chung một VM "Database" (chạy 3 service này dưới dạng 3 container riêng biệt qua Docker Compose, xem [[05-Labs]]) là một đánh đổi hợp lý giữa việc mô phỏng đúng kiến trúc phân tầng và giới hạn tài nguyên máy lab thực tế của học viên — bạn vẫn luyện được đầy đủ kỹ năng network/volume cho từng service, chỉ khác là chúng chia sẻ chung tài nguyên vật lý của một VM.
