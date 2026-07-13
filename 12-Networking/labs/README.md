---
title: "Labs - Module 12"
module: 12
tags: [docker, sysops-infra, module-12, labs]
---

# Labs - Module 12: Networking

Thư mục lưu kết quả thực hành từ [[../05-Labs|05-Labs.md]].

## Môi trường

- VM Linux đã cài Docker Engine 29.x, quyền `sudo` để chạy `iptables`, `nsenter`, `sysctl`.
- Lab 5 (macvlan) cần biết chính xác subnet/interface vật lý của VM và một máy khác trong cùng LAN để test ping từ ngoài vào.

## File dự kiến sinh ra trong thư mục này

| File | Sinh ra ở Lab | Nội dung |
|---|---|---|
| `dns-comparison.md` | Lab 1 | Bằng chứng default bridge vs user-defined bridge (output ping, resolv.conf) |
| `iptables-evidence.md` | Lab 2 | Rule DNAT chụp lại trước/sau khi publish port |
| `masquerade-notes.md` | Lab 3 | Bằng chứng MASQUERADE, ảnh hưởng của `ip_forward` |
| `host-network-notes.md` | Lab 4 | So sánh `ip addr` giữa host network và bridge network |
| `macvlan-setup.md` | Lab 5 | Cấu hình macvlan, bằng chứng ping từ LAN thành công, ping từ host thất bại |
| `appnet-diagram.md` | Lab 6 | Sơ đồ ASCII 3 container (web/backend/db), ghi chú thiết kế |

## Lưu ý an toàn

- Không tắt `ip_forward` hoặc thử nghiệm firewall trên server đang phục vụ traffic thật — chỉ thực hiện trên VM lab riêng.
- Lab macvlan có thể không hoạt động trên một số nền tảng cloud/VM lồng nhau do NIC không cho phép nhiều MAC — nếu gặp lỗi, ghi chú lại và thử trên VM khác hỗ trợ promiscuous mode thay vì cố ép chạy trên môi trường không phù hợp.
