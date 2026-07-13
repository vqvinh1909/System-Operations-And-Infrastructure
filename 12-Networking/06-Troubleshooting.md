---
title: "06 - Troubleshooting"
module: 12
tags: [docker, sysops-infra, module-12, troubleshooting]
---

# 06 - Troubleshooting: Networking

## Sự cố 1: Container A không resolve được tên container B

**Chẩn đoán**:
```bash
docker exec containerA cat /etc/resolv.conf
docker network inspect $(docker inspect containerA --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}{{end}}')
```

**Nguyên nhân gốc**: cả hai container đang nằm trên **default bridge** (`docker0`) — network này không có embedded DNS server. Nếu `/etc/resolv.conf` không có `127.0.0.11`, đây chính xác là nguyên nhân.

**Khắc phục**: tạo user-defined network (`docker network create mynet`), gắn lại cả hai container vào đó (`docker network connect mynet containerA` hoặc chạy lại container với `--network mynet`).

## Sự cố 2: `docker run -p 8080:80` báo lỗi `port is already allocated`

**Chẩn đoán**:
```bash
docker ps --filter "publish=8080"
sudo ss -tlnp | grep 8080
```

**Nguyên nhân**: một container khác (còn tồn tại dù đã `exited` trong một số trường hợp cấu hình) hoặc một tiến trình khác trên host đã chiếm port đó. Docker không tự phát hiện xung đột với tiến trình ngoài Docker cho tới khi thực sự cố bind.

**Khắc phục**: xác định đúng thủ phạm bằng `ss -tlnp`, dừng/xóa container cũ đang giữ port (`docker rm -f <container>`), hoặc chọn port khác nếu tiến trình ngoài Docker cần giữ nguyên.

## Sự cố 3: Container không ra được Internet dù network cấu hình đúng

**Chẩn đoán theo thứ tự**:
```bash
docker exec <container> ping -c 2 8.8.8.8      # test tang IP thuan tuy (loai DNS)
sysctl net.ipv4.ip_forward                      # phai = 1
sudo iptables -t nat -L POSTROUTING -n           # tim rule MASQUERADE
```

**Nguyên nhân thường gặp**: (1) `net.ipv4.ip_forward=0` bị một công cụ khác (firewall, hardening script) tắt sau khi Docker đã bật lúc cài đặt; (2) một firewall khác (`firewalld`, `ufw`) được bật sau khi Docker cài, ghi đè/chèn rule làm gián đoạn chain `DOCKER`/`POSTROUTING` mà Docker đã thiết lập; (3) container dùng `--network none`.

**Khắc phục**: bật lại `ip_forward` (`sysctl -w net.ipv4.ip_forward=1`, và persist trong `/etc/sysctl.conf` hoặc `/etc/sysctl.d/`), kiểm tra thứ tự load firewall — khuyến nghị chính thức là cấu hình firewall tương thích với Docker (nhiều tổ chức dùng `iptables` thuần thay vì để `firewalld` quản lý cùng lúc để tránh xung đột), hoặc restart Docker daemon sau khi thay đổi cấu hình firewall để nó thiết lập lại rule của mình.

## Sự cố 4: Macvlan container không ping được chính host Docker

**Đây không phải lỗi cấu hình** — là giới hạn thiết kế của kernel Linux: traffic từ macvlan sub-interface không được host "nhìn thấy" trực tiếp qua interface cha của nó, do cách macvlan tách biệt namespace ở tầng driver.

**Giải pháp**: tạo thêm một macvlan sub-interface **trên chính host**, cùng subnet với macvlan network của container, làm "cầu nối" — host giao tiếp với container macvlan thông qua chính sub-interface này thay vì interface vật lý gốc.

## Sự cố 5: Port `EXPOSE` trong Dockerfile nhưng client bên ngoài không truy cập được

**Nguyên nhân**: hiểu nhầm `EXPOSE` tự động mở port. `EXPOSE` chỉ là metadata/documentation, không tạo bất kỳ rule iptables nào.

**Khắc phục**: phải publish port tường minh bằng `-p host_port:container_port` (hoặc `-P` để publish tất cả port đã EXPOSE ra port ngẫu nhiên) khi `docker run`.

## Sự cố 6: Sau khi restart Docker daemon, một số container mất kết nối mạng tạm thời dù vẫn `Up`

**Nguyên nhân**: `dockerd` khi khởi động lại có thể cần rebuild lại các rule iptables (đặc biệt nếu có công cụ khác can thiệp vào bảng `nat` trong lúc daemon down). Với container dùng bridge network, IP/veth thường được giữ nguyên (không ảnh hưởng do kiến trúc containerd-shim độc lập đã học ở Module 09), nhưng rule iptables publish port có thể cần thời gian ngắn để daemon thiết lập lại đầy đủ ngay sau khi start.

**Chẩn đoán**: `sudo iptables -t nat -L DOCKER -n` ngay sau khi daemon restart, xác nhận rule DNAT cho từng container publish port đã được tái tạo đầy đủ.

**Khắc phục/phòng ngừa**: tránh restart `dockerd` bừa bãi trên node production trong giờ cao điểm; nếu bắt buộc, theo dõi ngay sau đó bằng health check ở tầng ứng dụng/load balancer để phát hiện sớm nếu có publish port nào chưa được khôi phục đúng.
