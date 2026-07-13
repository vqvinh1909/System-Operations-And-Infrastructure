---
title: "01 - Introduction"
module: 12
tags: [docker, sysops-infra, module-12, introduction]
---

# 01 - Introduction

## Bối cảnh: vì sao container cần một chương riêng cho networking

Ở Module 11, Vinh đã học container là gì và container runtime (containerd, runc) khởi động container ra sao. Nhưng một container chạy đơn lẻ, không giao tiếp được với ai, gần như vô dụng trong môi trường doanh nghiệp thực tế:

- Một container web server (nginx) cần expose port 80/443 ra ngoài để người dùng truy cập.
- Một container ứng dụng (backend) cần gọi được container database (PostgreSQL) — thường qua tên container, không phải IP cứng vì IP container có thể đổi mỗi lần restart.
- Một container batch job xử lý dữ liệu nội bộ có thể **không** cần network ra ngoài — càng ít network access càng giảm bề mặt tấn công (attack surface).
- Một hệ thống chạy nhiều host (Docker Swarm cluster) cần các container trên các máy vật lý khác nhau nói chuyện được với nhau như thể chúng ở chung một mạng LAN ảo.

Mỗi tình huống trên ứng với một **network driver** khác nhau của Docker: bridge, host, none, overlay, macvlan. Chọn sai driver không chỉ gây lỗi kỹ thuật mà còn là lỗi kiến trúc — ví dụ dùng `host` network cho một container chạy nhiều instance trên cùng port sẽ đụng port ngay lập tức; dùng `bridge` mặc định (default bridge) thay vì user-defined bridge sẽ khiến hai container không resolve được tên của nhau, một lỗi cực kỳ phổ biến khi người mới bắt đầu container hoá ứng dụng nhiều tầng (multi-tier app).

## Tại sao một Sysadmin cần hiểu sâu — không chỉ "biết dùng lệnh"

Trong thực tế vận hành, networking là nơi xảy ra nhiều sự cố nhất khi làm việc với Docker, vì nó nằm ở giao điểm giữa:

1. **Container** (namespace, cấu hình DNS resolver bên trong container).
2. **Docker daemon** (quản lý bridge, veth pair, cấu hình iptables/nftables).
3. **Hệ điều hành host** (routing table, firewall, kernel networking — đúng những gì Vinh đã học ở khóa Linux Sysadmin).
4. **Hạ tầng mạng doanh nghiệp** (firewall công ty, VLAN, DNS nội bộ — với driver macvlan hoặc khi publish port ra ngoài).

Một Sysadmin/DevOps không hiểu tầng dưới sẽ chỉ biết gõ `docker run -p 8080:80` và cầu mong nó chạy. Một Sysadmin hiểu sâu sẽ biết: lệnh đó tạo ra một rule iptables DNAT, biết rule đó nằm ở chain nào, biết cách `iptables -t nat -L -n` để xác nhận, và biết phải sửa gì khi rule không được tạo ra (ví dụ do dùng nftables thuần và Docker cấu hình sai, hoặc do một tool bên thứ ba như firewalld ghi đè rule của Docker).

## Mục tiêu học tập

Sau khi hoàn thành module này, Vinh sẽ:

1. Giải thích được vì sao Docker network thực chất là Linux network namespace + veth pair + bridge + iptables/nftables NAT, không phải công nghệ độc quyền của Docker.
2. Phân biệt được 5 network driver: bridge, host, none, overlay, macvlan — biết khi nào dùng cái nào, đánh đổi (trade-off) của từng loại.
3. Giải thích được vì sao **user-defined bridge network** tốt hơn **default bridge** (do có embedded DNS server tại 127.0.0.11), và tự thiết lập network cho nhiều container giao tiếp qua tên.
4. Phân biệt "published" (`-p`/`--publish`) và "exposed" (`--expose`/`EXPOSE`) port — hiểu cơ chế port mapping hoạt động qua iptables DNAT.
5. Thao tác thành thạo `docker network create/ls/inspect/connect/disconnect/rm`.
6. Tự troubleshoot được các lỗi networking thường gặp: container không resolve tên nhau, port đã bị chiếm, container không ra Internet, macvlan không ping được host.

## Cấu trúc module

Module đi theo trình tự: lý thuyết + internal working ([[02-Theory]]) → sơ đồ kiến trúc để hình dung trực quan ([[03-Architecture]]) → lệnh cụ thể ([[04-Commands]]) → thực hành từ cơ bản đến nâng cao ([[05-Labs]]) → xử lý sự cố thực tế ([[06-Troubleshooting]]) → ôn tập phỏng vấn ([[07-Interview]]) → tổng kết ([[08-Summary]]).

Khuyến nghị: đọc lý thuyết trước, nhưng đừng chỉ đọc suông — mở một máy có Docker (VM hoặc máy thật), vừa đọc [[02-Theory]] vừa gõ lệnh kiểm chứng ngay (`ip addr`, `iptables -t nat -L -n`, `docker network inspect`). Networking là chủ đề học bằng tay nhanh hơn học bằng mắt.
