---
title: "05 - Labs"
module: 12
tags: [docker, sysops-infra, module-12, labs, hands-on]
---

# 05 - Labs: Networking

Môi trường: VM Linux đã cài Docker Engine 29.x, quyền `sudo` để chạy `iptables`/`nsenter` kiểm chứng. Lưu kết quả vào [[labs/README|labs/]].

## Lab 1 (Basic): Default bridge vs User-defined bridge — chứng minh DNS

1. Chạy 2 container trên default bridge (không chỉ định `--network`): `docker run -d --name db1 nginx:1.27-alpine`, `docker run -d --name app1 nginx:1.27-alpine`.
2. Thử `docker exec app1 ping -c 2 db1` — quan sát lỗi resolve tên (không tìm thấy host).
3. Xác nhận bằng IP vẫn ping được: lấy IP `db1` qua `docker inspect db1 --format '{{.NetworkSettings.IPAddress}}'`, ping bằng IP đó — thành công.
4. Tạo user-defined network: `docker network create labnet`, chạy lại 2 container mới gắn `--network labnet`.
5. Lặp lại `ping` bằng tên — lần này thành công. Ghi lại `docker exec <container> cat /etc/resolv.conf` để thấy `127.0.0.11`.

## Lab 2 (Basic): Publish vs Expose — quan sát bằng iptables

1. Chạy container không publish port: `docker run -d --name web1 nginx:1.27-alpine`, thử `curl localhost:80` từ host — thất bại.
2. Chạy container có publish: `docker run -d --name web2 -p 8080:80 nginx:1.27-alpine`, thử `curl localhost:8080` — thành công.
3. Chạy `sudo iptables -t nat -L DOCKER -n --line-numbers` — tìm đúng rule DNAT tương ứng `web2`, xác nhận `web1` không có rule nào.
4. Xóa `web2` (`docker rm -f web2`), chạy lại lệnh iptables ở bước 3 — xác nhận rule DNAT đã tự động bị gỡ.

## Lab 3 (Intermediate): Container ra Internet — theo dõi MASQUERADE

1. Chạy container trên user-defined network: `docker run -it --rm --network labnet alpine sh`.
2. Bên trong, `ping -c 3 8.8.8.8` và `wget -qO- https://example.com | head -5` — xác nhận ra Internet được.
3. Từ host, chạy `sudo iptables -t nat -L POSTROUTING -n` — tìm rule `MASQUERADE` áp dụng cho subnet của `labnet`.
4. Kiểm tra `sysctl net.ipv4.ip_forward` — xác nhận giá trị là `1` (điều kiện bắt buộc để traffic đi qua được).
5. Tạm tắt `ip_forward` (`sudo sysctl -w net.ipv4.ip_forward=0`) trong VM lab, thử lại bước 2 — quan sát traffic không còn ra được Internet, rồi bật lại (`=1`) để khôi phục.

## Lab 4 (Intermediate): `--network host` — hiểu đánh đổi

1. Chạy container đơn giản lắng nghe port 8080 với `--network host`.
2. Xác nhận container KHÔNG cần `-p` vẫn truy cập được qua `localhost:8080` từ host (vì dùng chung netns).
3. Thử chạy thêm một container khác cũng cố lắng nghe port 8080 với `--network host` — quan sát lỗi "address already in use", giải thích vì sao (không có NAT/port mapping để tách biệt).
4. So sánh `docker exec <container> ip addr` giữa container `host` network và container `bridge` network thông thường — ghi lại khác biệt quan sát được.

## Lab 5 (Advanced): Macvlan — container xuất hiện như thiết bị LAN thật

**Yêu cầu**: cần biết chính xác subnet/gateway/interface vật lý của VM lab (thường `eth0` hoặc tương tự), và VM lab phải cho phép chế độ promiscuous/nhiều MAC trên NIC (kiểm tra trước với hạ tầng ảo hóa nếu chạy trên cloud/VM lồng nhau, vì một số nền tảng chặn macvlan theo mặc định).

1. Xác định subnet thật của VM (`ip addr show eth0`), tạo macvlan network đúng subnet đó.
2. Chạy container gắn vào macvlan network, gán IP tĩnh trong cùng subnet nhưng ngoài dải DHCP đang cấp phát (tránh xung đột).
3. Từ một máy khác trong cùng mạng LAN (không phải host chạy Docker), thử `ping` vào IP của container — xác nhận container "xuất hiện" như một thiết bị độc lập.
4. Từ chính host, thử `ping` container đó — quan sát và ghi lại việc **không ping được** (giới hạn kernel của macvlan), đối chiếu với lý thuyết ở [[02-Theory]] mục 4.4.

## Lab 6 (Advanced/Mini Project): Mô hình 3 container giao tiếp qua user-defined network

**Mục tiêu**: dựng một kiến trúc web + backend + database tối giản, chỉ dùng `docker run` + `docker network` (chưa dùng Compose — sẽ học ở Module 14), củng cố toàn bộ kiến thức DNS + port mapping.

1. Tạo network `appnet`.
2. Chạy container `db` (ví dụ `postgres:16-alpine`, đặt `--network appnet`, không publish port ra ngoài — chỉ cần nội bộ).
3. Chạy container `backend` (image tối giản tự viết, kết nối tới `db` bằng tên host `db`, không publish port ra ngoài).
4. Chạy container `frontend`/`nginx` publish port `-p 8080:80`, cấu hình reverse proxy trỏ tới `backend` bằng tên.
5. Xác nhận toàn bộ chuỗi hoạt động: `curl localhost:8080` từ host phải đi xuyên qua `nginx` → `backend` → `db` và trả về đúng dữ liệu.
6. Vẽ lại (bằng ASCII, tay hoặc ghi chú) sơ đồ toàn bộ luồng gói tin qua 3 container, lưu vào `labs/appnet-diagram.md`.

**Kết quả cần đạt**: chứng minh hiểu đúng nguyên tắc chỉ publish port ở tầng cần thiết nhất (chỉ `nginx` cần `-p`, `backend`/`db` giao tiếp nội bộ qua DNS, không publish ra ngoài) — đúng nguyên tắc giảm bề mặt tấn công trong thiết kế network doanh nghiệp.
