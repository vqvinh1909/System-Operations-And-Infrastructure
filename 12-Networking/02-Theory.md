---
title: "02 - Theory"
module: 12
tags: [docker, sysops-infra, module-12, theory, internal-working]
---

# 02 - Theory

## 1. Nền tảng: Docker network dùng lại đúng những gì Linux kernel đã có

Trước khi vào từng driver, cần khẳng định một điều: **Docker không tự viết ra bộ network stack riêng**. Docker daemon (`dockerd`) chỉ là một chương trình gọi các API và công cụ Linux đã tồn tại từ lâu:

| Khái niệm Docker hay nhắc | Thực chất là gì trong Linux (Vinh đã học ở khóa Sysadmin) |
|---|---|
| "Container có network riêng" | Một network namespace (`netns`) mới, tạo bằng syscall `clone()`/`unshare(CLONE_NEWNET)` |
| "docker0 bridge" | Một Linux bridge (`brctl`/`ip link add type bridge`), y hệt bridge Vinh từng tạo thủ công |
| "Container kết nối vào bridge" | Một cặp `veth` (virtual ethernet pair) — một đầu nhét vào netns container, một đầu gắn vào bridge trên host |
| "Port mapping -p 8080:80" | Một rule iptables/nftables trong bảng `nat`, chain `DOCKER` (được `dockerd` chèn vào chain `PREROUTING`/`OUTPUT`) |
| "Container ra được Internet" | Rule `MASQUERADE` (SNAT động) trong chain `POSTROUTING`, cộng với `ip_forward=1` ở kernel |

Nói cách khác: khi gõ `docker run`, `dockerd` làm hộ Vinh một loạt thao tác mà nếu không có Docker, Vinh vẫn có thể tự làm bằng tay với `ip netns`, `ip link`, `iptables` — chỉ là chậm và dễ sai hơn nhiều. Hiểu được điều này giúp việc troubleshooting không còn là "đoán mò lệnh Docker" mà quay về đúng kỹ năng nền Vinh đã vững: đọc bridge, đọc route, đọc iptables.

## 2. Internal Working: chuyện gì xảy ra khi `docker run` với network bridge

Khi chạy `docker run -d --name web --network mynet -p 8080:80 nginx`, theo đúng trình tự bên trong:

1. **Tạo network namespace mới** cho container (nếu chưa có sẵn qua `--network container:...`). Namespace này cô lập hoàn toàn bảng interface, routing table, iptables rule... của container khỏi host.
2. **Tạo cặp veth**: `veth<hash1>` ở phía host và một interface (thường đặt tên `eth0`) ở phía trong netns của container. Veth pair hoạt động như một sợi cáp mạng ảo nối 2 đầu — gói tin gửi vào một đầu sẽ xuất hiện ngay ở đầu kia.
3. **Gắn đầu host của veth vào bridge** (`mynet` — hoặc `docker0` nếu dùng default bridge). Từ lúc này, container về mặt Layer 2 giống hệt một máy cắm dây mạng vào switch ảo là bridge đó.
4. **Gán IP cho interface trong container** (ví dụ `172.18.0.2/16`), đặt default gateway là IP của chính bridge (ví dụ `172.18.0.1` — đây là lý do container luôn "ping được gateway" chính là ping bridge trên host).
5. **Ghi cấu hình DNS** vào `/etc/resolv.conf` bên trong container: nếu là user-defined network, nameserver sẽ là `127.0.0.11` (embedded DNS — nói chi tiết ở mục 5).
6. **Chèn rule iptables DNAT** cho phần `-p 8080:80`: gói tin đến host ở port 8080 sẽ được NAT thành đích `172.18.0.2:80` (nói chi tiết ở mục 6).

Toàn bộ quá trình này Vinh có thể xem trực quan ở [[03-Architecture]].

## 3. Bridge network — driver mặc định

### 3.1. Default bridge (`docker0`)

Khi cài Docker, daemon tự tạo sẵn một bridge tên `docker0` với dải IP mặc định (thường `172.17.0.0/16`). Bất kỳ container nào chạy `docker run` **không chỉ định** `--network` sẽ tự động nối vào `docker0`.

Vấn đề của default bridge — và đây là lý do gần như mọi tài liệu Docker production đều khuyến cáo **không dùng default bridge**:

- **Không có embedded DNS**. Container trên default bridge không resolve được tên container khác. Muốn giao tiếp phải biết IP (mà IP container có thể đổi sau mỗi lần restart) hoặc dùng `--link` (đã deprecated từ lâu, không nên dùng cho hệ thống mới).
- **Không cô lập theo ứng dụng**. Tất cả container không chỉ định network đều rơi vào chung một bridge — hai dự án khác nhau chạy trên cùng host vô tình "thấy" nhau ở Layer 2, tăng rủi ro bảo mật.

### 3.2. User-defined bridge — best practice

Khi chạy `docker network create mynet` rồi `docker run --network mynet ...`, Docker tạo một bridge mới (không phải `docker0`), có DNS server riêng cho network đó. Lợi ích:

- **Embedded DNS hoạt động đầy đủ**: container resolve được tên nhau (chi tiết mục 5).
- **Cô lập tốt hơn**: chỉ container cùng khai báo `--network` mới nằm chung bridge, thấy nhau ở Layer 2.
- **Có thể connect/disconnect linh hoạt lúc container đang chạy** (`docker network connect`/`disconnect`), default bridge không hỗ trợ linh hoạt như vậy.
- **Hỗ trợ network alias**, cho phép một container có nhiều tên gọi khác nhau trong cùng network — hữu ích khi swap phiên bản service mà không đổi tên gọi ở phía client.

> Nguyên tắc thực chiến: **luôn tạo user-defined bridge network cho mọi ứng dụng multi-container**, kể cả khi chạy trên một host đơn lẻ bằng `docker run` thuần (chưa dùng Compose). Compose tự động tạo user-defined network cho bạn — đây cũng là lý do "Compose tự nhiên hoạt động DNS theo tên service" mà nhiều người dùng Compose không để ý lý do tại sao.

## 4. Host, None, Overlay, Macvlan

### 4.1. Host network

`--network host`: container **dùng chung network namespace với host**, không tạo netns riêng, không có bridge, không có veth pair, không NAT.

- Container thấy đúng interface, đúng IP, đúng routing table như host.
- Port container lắng nghe (ví dụ port 80) chính là port trên host — **không cần** `-p`, và **không thể** dùng `-p` (không có gì để map).
- Hệ quả: không thể chạy 2 container cùng lắng nghe port 80 trên host network — đụng port ngay (`bind: address already in use`), y hệt chạy 2 process thường trên host.
- Dùng khi: cần hiệu năng mạng tối đa (bỏ qua overhead NAT/bridge), ví dụ công cụ giám sát mạng cần thấy traffic thật của host, hoặc ứng dụng cực nhạy về latency.
- Lưu ý bảo mật: container có network namespace = host nghĩa là nếu container bị compromise, kẻ tấn công có góc nhìn mạng gần như đầy đủ của host.

### 4.2. None network

`--network none`: container có netns riêng nhưng **chỉ có loopback (`lo`)**, không có interface nào khác, không bridge, không veth ra ngoài.

- Container hoàn toàn cô lập về mạng — không nhận, không gửi được gói tin ra ngoài (trừ khi tự tay thêm interface sau bằng công cụ như `pipework`/thủ công với `ip netns`).
- Dùng khi: batch job xử lý dữ liệu cục bộ, không cần mạng — giảm tối đa bề mặt tấn công. Ví dụ: container chạy script nén file, tính toán offline, không có lý do gì để nó có thể gọi ra Internet.

### 4.3. Overlay network (khái niệm — nền tảng cho Swarm/multi-host)

Khóa này không đi sâu Docker Swarm, nhưng Vinh cần biết khái niệm vì kiến trúc tương tự xuất hiện lại ở Kubernetes (CNI overlay) sau này.

- Overlay network cho phép container trên **nhiều Docker host khác nhau** giao tiếp như thể chúng nằm chung một mạng Layer 2, dù thực tế các host có thể ở các subnet khác nhau, thậm chí khác datacenter.
- Cơ chế: đóng gói (encapsulate) gói tin container vào một gói **VXLAN** (Virtual Extensible LAN) rồi gửi qua mạng vật lý thật (Layer 3) giữa các Docker host. Bên nhận giải gói (decapsulate) VXLAN để lấy lại gói tin gốc rồi đưa vào container đích.
- Overlay network yêu cầu Docker chạy ở **Swarm mode**, cần mở các cổng: `2377/tcp` (quản lý cluster), `7946/tcp+udp` (discovery giữa các node), `4789/udp` (VXLAN data plane).
- Overlay cũng có embedded DNS riêng cho service trong Swarm — nguyên lý giống DNS của user-defined bridge nhưng mở rộng ra nhiều host.
- Liên hệ thực tế: nếu công ty triển khai Docker Swarm cluster nhiều node, và service A ở node 1 gọi service B ở node 2 bằng tên service — chính overlay network + VXLAN đang âm thầm đóng gói/giải gói traffic đó.

### 4.4. Macvlan

Macvlan cho phép container có **địa chỉ MAC và IP riêng, xuất hiện trên mạng LAN vật lý như một thiết bị độc lập** — không qua NAT, không qua bridge của Docker.

- Cơ chế: interface macvlan "nhân bản" (virtualize) một interface vật lý của host (ví dụ `eth0`), tạo ra các sub-interface ảo, mỗi sub-interface có MAC riêng, gắn thẳng vào container.
- Vì có IP thật trong dải mạng LAN, container **giao tiếp trực tiếp** với các thiết bị khác trong mạng công ty (máy chủ khác, switch, thiết bị legacy) mà không cần port mapping.
- **Giới hạn quan trọng của kernel Linux**: container trên macvlan network **không tự ping/giao tiếp được với chính host** (host không thấy được traffic từ macvlan interface do nó tạo ra, đây là giới hạn thiết kế của macvlan trong kernel, không phải bug). Muốn container macvlan giao tiếp với host, cách phổ biến là tạo thêm một macvlan sub-interface **trên chính host** ở cùng subnet để làm cầu nối.
- Dùng khi: có ứng dụng legacy cần một IP cố định "thấy được" trực tiếp trên mạng LAN doanh nghiệp (ví dụ hệ thống giám sát mạng theo IP, thiết bị nhận diện theo IP cố định, hoặc di trú (migrate) một service từ VM vật lý sang container mà không muốn đổi kiến trúc mạng).
- Đánh đổi: switch công ty cần cho phép nhiều MAC trên cùng cổng vật lý (một số switch bật "port security" giới hạn số MAC/cổng sẽ chặn macvlan) — đây là điều cần trao đổi trước với đội network khi triển khai.

## 5. DNS: Docker embedded DNS server (127.0.0.11)

### 5.1. Cách hoạt động

Với mọi container nằm trên **user-defined network** (bridge, overlay), Docker daemon tự cấu hình `/etc/resolv.conf` bên trong container với:

```
nameserver 127.0.0.11
```

`127.0.0.11` không phải là một service chạy trên host — nó là **một DNS resolver ảo tồn tại bên trong network namespace của chính container đó**, do `dockerd` cấy vào bằng iptables/netfilter (chuyển hướng truy vấn tới `127.0.0.11:53` vào tiến trình DNS resolver nội bộ của Docker). Vì vậy `docker exec <container> cat /etc/resolv.conf` sẽ luôn thấy `127.0.0.11` bất kể chạy trên host nào.

### 5.2. Quy trình resolve tên

1. Container A truy vấn tên "db" → gửi tới `127.0.0.11:53` (không đi ra ngoài Internet).
2. Docker daemon tra trong **bảng ánh xạ tên nội bộ của network đó** — bảng này được daemon cập nhật tự động mỗi khi có container connect/disconnect khỏi network (không cần Vinh tự sửa file hosts).
3. Nếu tìm thấy (ví dụ container tên "db" hoặc có network alias "db"), trả về IP nội bộ (ví dụ `172.18.0.3`).
4. Nếu **không** tìm thấy trong bảng nội bộ, Docker forward truy vấn ra DNS server thật (mặc định kế thừa DNS resolver của host, hoặc DNS chỉ định qua `--dns` khi tạo container) để resolve tên miền ra Internet bình thường.

### 5.3. Vì sao default bridge không có DNS này

Cơ chế embedded DNS gắn liền với **network cụ thể** (mỗi user-defined network có bảng ánh xạ tên riêng). Default bridge (`docker0`) là network "toàn cục" dùng chung cho mọi container không chỉ định driver — Docker không cấp DNS server riêng cho nó vì lý do lịch sử (tính năng ra đời sau, và default bridge được giữ nguyên hành vi cũ để không phá vỡ tương thích ngược). Đây chính là lý do kỹ thuật cốt lõi khiến "container không resolve được tên nhau" gần như luôn là dấu hiệu của việc quên tạo/chỉ định user-defined network.

## 6. Port Mapping: Published vs Exposed, và Internal Working qua iptables DNAT

### 6.1. Khác biệt publish và expose

| | `-p`/`--publish` (lúc `docker run`) | `--expose` / `EXPOSE` (trong Dockerfile hoặc `docker run`) |
|---|---|---|
| Mục đích | Mở cổng container ra **ngoài host**, client bên ngoài gọi được | Chỉ mang tính **documentation** — khai báo container lắng nghe cổng nào |
| Có tạo rule iptables không | **Có** — tạo rule DNAT thật | **Không** — không hề chạm iptables |
| Ai gọi được | Bất kỳ ai truy cập được `host_ip:port` | Chỉ container khác **trong cùng network** (và điều này xảy ra mặc định dù có `--expose` hay không — container cùng network luôn gọi thẳng nhau qua IP nội bộ/tên) |
| Ví dụ | `-p 8080:80` | `EXPOSE 80` trong Dockerfile |

Hiểu lầm phổ biến: nhiều người nghĩ `EXPOSE` trong Dockerfile "mở port" — thực ra nó **không mở gì cả**, chỉ là ghi chú cho người dùng image biết cần publish port nào nếu muốn truy cập từ bên ngoài. Container trong cùng network luôn gọi được nhau bất kể có `EXPOSE` hay không, vì bridge network không có khái niệm "port bị chặn giữa các container cùng subnet" trừ khi có thêm rule iptables/firewall chặn riêng.

### 6.2. Internal Working: `-p 8080:80` tạo ra gì trong iptables

Khi container chạy với `-p 8080:80`, Docker daemon chèn (không phải thay thế) các rule vào bảng `nat` của iptables (hoặc nftables tuỳ cấu hình hệ thống):

1. **Chain `DOCKER`** (nằm trong bảng `nat`) nhận một rule dạng:
   ```
   DNAT tcp -- 0.0.0.0/0  0.0.0.0/0  tcp dpt:8080  to:172.18.0.2:80
   ```
   Nghĩa là: gói tin TCP đến bất kỳ đích nào ở port 8080 sẽ bị đổi địa chỉ đích (Destination NAT) thành `172.18.0.2:80` — chính IP nội bộ của container.
2. Chain `DOCKER` được **gọi từ chain `PREROUTING`** (cho traffic đến từ ngoài) và chain `OUTPUT` (cho traffic sinh ra từ chính host, ví dụ host tự `curl localhost:8080`) — đây là lý do publish port hoạt động cả khi client ở máy khác lẫn khi gọi từ chính host.
3. Sau khi DNAT, gói tin được **route** vào bridge, gửi qua veth pair tới container — quá trình routing/forward này cần `net.ipv4.ip_forward=1` ở kernel (Docker tự bật khi cài đặt, đây cũng là điểm cần kiểm tra khi troubleshoot container không nhận được traffic dù rule DNAT đã đúng).
4. Chiều ngược lại, khi container trả lời, gói tin đi qua rule liên quan trong chain `POSTROUTING` để đảm bảo địa chỉ nguồn trả về đúng client ban đầu (nhờ kernel theo dõi connection tracking — `conntrack`), không phải NAT thủ công từng chiều.

Có thể tự kiểm chứng bằng:
```
sudo iptables -t nat -L DOCKER -n --line-numbers
```
Sẽ thấy đúng rule DNAT tương ứng với mỗi container đang publish port — đây là kỹ năng troubleshooting cốt lõi, chi tiết áp dụng ở [[06-Troubleshooting]].

## 7. Tóm tắt liên kết Linux networking mà Vinh đã biết

| Kỹ năng Linux Sysadmin đã học | Áp dụng trực tiếp vào Docker networking |
|---|---|
| `ip netns` | Mỗi container = một network namespace (`docker inspect` cho thấy đường dẫn netns) |
| Linux bridge, `brctl`/`ip link` | `docker0` và mọi user-defined bridge chính là Linux bridge thật |
| `veth` pair | Cách container "cắm dây" vào bridge |
| `iptables`/`nftables`, bảng `nat` | Cơ chế publish port (DNAT) và container ra Internet (MASQUERADE) |
| `ip_forward`, routing table | Điều kiện cần để traffic đi xuyên qua bridge tới container |
| `/etc/resolv.conf`, DNS resolver | Cách container tìm nameserver `127.0.0.11` |

Xem tiếp [[03-Architecture]] để hình dung trực quan bằng sơ đồ ASCII trước khi thực hành lệnh ở [[04-Commands]].
