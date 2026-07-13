---
title: "08 - Summary"
module: 12
tags: [docker, sysops-infra, module-12, summary]
---

# 08 - Summary: Networking

## Những gì đã học

- **Docker không phát minh công nghệ mạng riêng** — nó gọi lại network namespace, Linux bridge, veth pair, iptables/nftables NAT mà Sysadmin đã biết từ khóa Linux cơ bản.
- **5 network driver**: `bridge` (mặc định, phù hợp đa số), `host` (hiệu năng cao, đánh đổi cô lập), `none` (cô lập tối đa), `overlay` (multi-host, VXLAN, cần Swarm), `macvlan` (container như thiết bị LAN thật).
- **User-defined bridge luôn tốt hơn default bridge** — có embedded DNS (`127.0.0.11`), cô lập tốt hơn, hỗ trợ connect/disconnect linh hoạt và network alias.
- **Publish vs Expose**: `-p` thực sự mở port qua iptables DNAT; `EXPOSE` chỉ là documentation, không mở gì cả.
- **Internal Working port mapping**: chain `PREROUTING`/`OUTPUT` → chain `DOCKER` (rule DNAT) → route qua bridge → veth → container; chiều về nhờ `conntrack`.
- **Macvlan có giới hạn kernel**: container không tự ping được host của nó, cần macvlan sub-interface trên host làm cầu nối.

## Checklist tự đánh giá

- [ ] Giải thích được vì sao default bridge không có DNS, user-defined bridge thì có.
- [ ] Phân biệt chính xác `-p` và `EXPOSE`, giải thích cơ chế DNAT bên dưới.
- [ ] Chọn đúng driver cho từng tình huống: web app thông thường, batch job không cần mạng, service cần hiệu năng mạng tối đa, thiết bị cần IP LAN cố định.
- [ ] Tự soi được rule iptables DNAT/MASQUERADE bằng `iptables -t nat -L`.
- [ ] Troubleshoot được 3 lỗi phổ biến nhất: container không resolve tên nhau, port already allocated, container không ra Internet.

## Liên kết tới Module tiếp theo

Container đã biết cách giao tiếp với nhau qua network. Nhưng dữ liệu bên trong container (writable layer) sẽ **mất khi container bị xóa** — [[../13-Storage/README|Module 13 - Storage]] giải quyết vấn đề dữ liệu bền vững: bind mount, named volume, tmpfs, và quy trình backup/restore dữ liệu container qua container tạm. Đây là mảnh ghép cuối cùng trước khi Module 14 (Docker Compose) tổng hợp network + storage + nhiều container vào một file cấu hình duy nhất.

> [!tip] Ghi nhớ cốt lõi
> Mọi sự cố networking Docker "kỳ lạ" đều có thể quy về đúng những gì bạn đã học ở Linux Sysadmin: network namespace, bridge, veth, iptables. Không có phép thuật nào trong Docker networking — chỉ là tự động hóa những thao tác bạn hoàn toàn có thể tự tay làm và kiểm chứng.
