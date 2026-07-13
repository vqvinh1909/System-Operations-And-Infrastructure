---
title: "07 - Interview"
module: 12
tags: [docker, sysops-infra, module-12, interview]
---

# 07 - Interview: Networking

## Junior

**1. Docker có bao nhiêu network driver mặc định? Kể tên.**
5 driver: `bridge` (mặc định), `host`, `none`, `overlay` (Swarm/multi-host), `macvlan`.

**2. Vì sao container trên default bridge không resolve được tên nhau, nhưng trên user-defined bridge thì được?**
Docker embedded DNS server (`127.0.0.11`) chỉ hoạt động trên user-defined network — mỗi network có bảng ánh xạ tên riêng, tự cập nhật khi container connect/disconnect. Default bridge (`docker0`) không có DNS server này vì lý do lịch sử/tương thích ngược.

**3. `-p 8080:80` và `EXPOSE 80` khác nhau thế nào?**
`-p`/`--publish` thực sự mở port ra ngoài host bằng cách tạo rule iptables DNAT — client bên ngoài truy cập được. `EXPOSE` chỉ là metadata/documentation trong Dockerfile, không tạo rule nào, không mở gì cả.

**4. `--network host` nghĩa là gì?**
Container dùng chung network namespace với host — không tạo netns riêng, không NAT, không cần (và không thể dùng) `-p`. Port container lắng nghe chính là port trên host.

**5. `--network none` dùng khi nào?**
Khi container không cần bất kỳ kết nối mạng nào ra ngoài (chỉ có loopback) — dùng cho batch job xử lý dữ liệu cục bộ, giảm tối đa bề mặt tấn công.

## Mid-level

**6. Giải thích Internal Working khi Docker tạo container trên bridge network — những gì kernel Linux thực sự làm.**
Tạo network namespace mới cho container, tạo cặp `veth` (một đầu trong netns container thường đặt tên `eth0`, đầu kia gắn vào bridge trên host), gán IP cho interface container với default gateway là IP của bridge, ghi `/etc/resolv.conf` trỏ `127.0.0.11` nếu là user-defined network, và chèn rule iptables DNAT nếu có publish port.

**7. Overlay network hoạt động ra sao ở tầng packet? Vì sao cần Swarm mode?**
Overlay đóng gói (encapsulate) gói tin container vào gói VXLAN, gửi qua mạng vật lý Layer 3 giữa các Docker host, bên nhận giải gói lấy lại gói tin gốc đưa vào container đích — tạo cảm giác các container trên nhiều host khác nhau nằm chung một Layer 2 ảo. Cần Swarm mode vì overlay network dựa vào cơ chế quản lý cluster (key-value store nội bộ, cổng 2377/7946/4789) mà chỉ Swarm mode khởi tạo và duy trì.

**8. Vì sao macvlan container không ping được chính host Docker của nó?**
Đây là giới hạn thiết kế của kernel Linux, không phải bug: traffic từ macvlan sub-interface không được host "nhìn thấy" trực tiếp qua interface cha mà nó bắt nguồn — cần tạo thêm một macvlan sub-interface riêng trên host, cùng subnet, làm cầu nối để host giao tiếp được với container macvlan.

**9. Giải thích chuỗi rule iptables khi publish port, gồm những chain nào?**
Chain `PREROUTING` (cho traffic từ ngoài vào) và `OUTPUT` (cho traffic sinh ra từ chính host) trong bảng `nat` gọi tới chain `DOCKER`, nơi chứa rule DNAT đổi địa chỉ đích thành IP nội bộ + port container. Sau khi route qua bridge/veth vào container, chiều trả lời dựa vào connection tracking (`conntrack`) của kernel, không cần NAT thủ công riêng cho chiều về. Chain `POSTROUTING` xử lý `MASQUERADE` cho traffic container ra Internet (khác với DNAT của publish port).

**10. `docker network connect`/`disconnect` cho phép làm gì mà `docker run --network` không làm được?**
Cho phép gắn/gỡ một container **đang chạy** vào/khỏi một network mà không cần restart hay tạo lại container — hữu ích khi cần tạm thời kết nối một container debug vào network production để chẩn đoán, hoặc khi một container cần thuộc nhiều network cùng lúc (ví dụ vừa nằm trong network nội bộ với database, vừa nằm trong network expose ra ngoài qua reverse proxy).

## Thực chiến

**11. Sau khi công ty bật `firewalld` trên toàn bộ server (theo yêu cầu compliance mới), các container Docker đột nhiên không publish port ra ngoài được nữa dù cấu hình `-p` không đổi. Chẩn đoán và hướng xử lý.**
`firewalld` khi bật thường flush hoặc quản lý lại bảng iptables/nftables theo cách riêng, có thể xóa hoặc chèn rule làm gián đoạn chain `DOCKER` mà Docker daemon đã thiết lập, hoặc chặn forwarding traffic vào bridge Docker theo policy mặc định của zone. Kiểm tra `sudo iptables -t nat -L DOCKER -n` xem rule DNAT còn tồn tại không; kiểm tra zone của interface Docker (`docker0`, bridge tự tạo) trong `firewall-cmd --get-zone-of-interface=docker0`. Hướng xử lý: cấu hình `firewalld` cho phép forwarding tới bridge Docker (thêm interface Docker vào zone `trusted` hoặc cấu hình rule tương thích), hoặc restart Docker daemon sau khi `firewalld` đã ổn định để nó tái thiết lập rule của mình đúng cách, và làm việc với team compliance để thống nhất một firewall layer duy nhất quản lý (tránh hai công cụ cùng thao tác bảng nat).

**12. Một service backend trong Kubernetes tương lai sẽ cần khái niệm network tương tự overlay — bạn giải thích cho team dev (chưa quen Docker networking) sự khác biệt giữa cách container trên 1 host giao tiếp (bridge) và cách container trên nhiều host giao tiếp (overlay) như thế nào để họ hiểu đúng bản chất, không chỉ nhớ tên?**
Trên 1 host, container giao tiếp qua bridge — về bản chất giống như cắm nhiều máy vào chung một switch ảo (Layer 2 thật, cùng subnet, cùng máy vật lý). Khi có nhiều host, không có switch vật lý chung nào nối trực tiếp các container ở Layer 2 — overlay network giải quyết bằng cách "giả lập" một switch ảo trải rộng qua nhiều host: mỗi gói tin container gửi đi được bọc thêm một lớp header VXLAN, gửi qua mạng Layer 3 thật (routing bình thường giữa các host) tới đúng host đích, rồi gỡ lớp bọc đó ra để container đích nhận đúng gói tin gốc như thể chưa từng rời khỏi mạng Layer 2 ban đầu — dev không cần quan tâm 2 container đang ở host nào, chỉ cần biết chúng "cùng nằm trên một mạng logic".

**13. Đội bảo mật yêu cầu một container xử lý dữ liệu nhạy cảm (PII) không được có bất kỳ khả năng kết nối Internet nào, kể cả vô tình do cấu hình sai sau này. Bạn thiết kế network cho container này thế nào, và làm sao đảm bảo tính bền vững của kiểm soát đó theo thời gian?**
Dùng `--network none` để container không có bất kỳ interface nào ngoài loopback — đây là kiểm soát ở tầng kernel (network namespace không có route ra ngoài), mạnh hơn nhiều so với việc chỉ dựa vào firewall rule (có thể bị sửa nhầm sau này). Để đảm bảo bền vững: đưa cấu hình `--network none` vào template triển khai chuẩn hóa (ví dụ trong Ansible playbook hoặc Compose file cố định, không cho phép override tùy tiện), và định kỳ audit bằng script kiểm tra `docker inspect --format '{{.HostConfig.NetworkMode}}'` trên toàn bộ container thuộc nhóm xử lý dữ liệu nhạy cảm để phát hiện sớm nếu ai đó vô tình đổi network mode khi cập nhật cấu hình.

**14. Một API gateway (nginx) cần cân bằng tải tới 3 instance của cùng một backend service, các instance này định kỳ được thay thế (rolling update) với IP nội bộ đổi liên tục. Giải pháp DNS/network nào của Docker giúp gateway luôn gọi đúng instance đang sống mà không cần sửa cấu hình nginx mỗi lần deploy?**
Dùng user-defined network với DNS round-robin: nếu nhiều container cùng đăng ký chung một network alias (hoặc dùng Docker DNS round-robin service name trong ngữ cảnh Swarm/Compose), truy vấn DNS tới tên đó trả về nhiều bản ghi IP luân phiên, để nginx (hoặc resolver runtime của nó) tự phân phối request giữa các instance đang tồn tại. Ở mức đơn giản hơn (không Swarm), có thể dùng `docker network connect --alias backend` cho mỗi container backend mới trước khi gỡ container cũ khỏi network — đảm bảo tại mọi thời điểm, tên `backend` luôn resolve ra ít nhất các instance đang thực sự chạy, không cần sửa file cấu hình nginx thủ công mỗi lần rolling update.
