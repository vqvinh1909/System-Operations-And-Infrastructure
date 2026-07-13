---
title: "03 - Architecture"
module: 12
tags: [docker, sysops-infra, module-12, architecture, diagram]
---

# 03 - Architecture: Luồng gói tin, So sánh Driver, DNS Resolution

Toàn bộ sơ đồ đã được đo bằng script Python (so sánh `len()` border trên/dưới và mọi dòng nội dung trong từng khung) trước khi đưa vào file.

## 1. Luồng gói tin: publish port `-p 8080:80` qua iptables DNAT

```text
+----------------------------------------------------+
|         Client ben ngoai goi host_ip:8080          |
+----------------------------------------------------+
                           |
                           v
+----------------------------------------------------+
|  Chain PREROUTING (bang nat) goi toi chain DOCKER  |
+----------------------------------------------------+
                           |
                           v
+----------------------------------------------------+
|        Rule DNAT: dpt 8080 -> 172.18.0.2:80        |
+----------------------------------------------------+
                           |
                           v
+----------------------------------------------------+
|          Route qua bridge (vi du docker0)          |
+----------------------------------------------------+
                           |
                           v
+----------------------------------------------------+
| veth pair chuyen goi tin sang eth0 trong container |
+----------------------------------------------------+
                           |
                           v
+----------------------------------------------------+
|      Container nhan goi tin that tai port 80       |
+----------------------------------------------------+
```

**Đọc sơ đồ**: gói tin từ client bên ngoài đi qua chain `PREROUTING` (bảng `nat`) trước khi kernel quyết định routing — đây là lý do NAT phải xảy ra "sớm", trước khi gói tin được route vào bridge nội bộ. Rule DNAT đổi địa chỉ đích thành IP nội bộ của container, sau đó gói tin đi qua đúng con đường vật lý-ảo: bridge → veth pair → interface bên trong container. Chiều trả lời đi ngược lại nhờ kernel connection tracking (`conntrack`), không cần NAT thủ công riêng cho chiều về.

## 2. So sánh 5 network driver

```text
+------------------------------------------------------------+
|                5 NETWORK DRIVER CUA DOCKER                 |
+------------------------------------------------------------+
|  bridge   - mac dinh, NAT qua docker0/user-defined bridge  |
+------------------------------------------------------------+
| host     - dung chung netns voi host, khong NAT, khong -p  |
+------------------------------------------------------------+
|   none     - chi co loopback, cach ly hoan toan ve mang    |
+------------------------------------------------------------+
| overlay  - noi nhieu Docker host qua VXLAN, can Swarm mode |
+------------------------------------------------------------+
|    macvlan  - container co MAC/IP rieng tren LAN vat ly    |
+------------------------------------------------------------+
```

Ghi nhớ: `bridge` là lựa chọn mặc định và phù hợp nhất cho đa số ứng dụng single-host; `host` đánh đổi cô lập lấy hiệu năng; `none` tối đa hóa cô lập; `overlay` chỉ có ý nghĩa khi có nhiều Docker host (Swarm); `macvlan` dùng khi cần container "trông giống" một thiết bị vật lý thật trên LAN.

## 3. Luồng DNS resolution qua embedded DNS (127.0.0.11)

```text
+----------------------------------------------------+
|     EMBEDDED DNS - LUONG RESOLVE TEN CONTAINER     |
+----------------------------------------------------+
|  Container A truy van ten "db" toi 127.0.0.11:53   |
+----------------------------------------------------+
|  Docker daemon tra bang anh xa ten cua network do  |
+----------------------------------------------------+
|      Neu co: tra ve IP noi bo (vd 172.18.0.3)      |
+----------------------------------------------------+
| Neu khong: forward truy van ra DNS that (Internet) |
+----------------------------------------------------+
```

**Điểm mấu chốt**: `127.0.0.11` không phải một tiến trình chạy trên host mà bạn có thể `ps aux` thấy trực tiếp — nó là một resolver ảo được `dockerd` cấy vào bên trong network namespace của từng container thông qua iptables/netfilter redirect. Cơ chế này chỉ tồn tại trên **user-defined network**, đây là lý do kỹ thuật cốt lõi khiến container trên default bridge (`docker0`) không resolve được tên nhau — không có bảng ánh xạ tên nào được tạo ra cho network đó.

Tiếp theo: [[04-Commands]] để xem cú pháp lệnh cụ thể áp dụng các khái niệm trên.
