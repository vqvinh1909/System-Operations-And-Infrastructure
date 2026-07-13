---
title: "05 - Labs"
module: 2
tags: [ansible, sysops-infra, module-02, labs]
---

# 05 — Labs thực hành

> Yêu cầu môi trường: control node (đã cài Ansible) + tối thiểu 3 managed node (2 giả lập web, 1 giả lập db), tái sử dụng SSH key/sudo đã chuẩn bị ở Module 01.

## Lab 1 — Cài đặt Control Node và xác nhận kết nối

**Mục tiêu:** có một control node hoạt động, kết nối được tới toàn bộ managed node.

**Các bước:**

1. Cài `ansible-core` trên control node, xác nhận bằng `ansible --version`.
2. Viết inventory tĩnh dạng YAML (`inventory.yml`) với 3 host: `web-01`, `web-02`, `db-01`, nhóm `webservers`, `dbservers`, và nhóm cha `production` gộp cả hai.
3. Chạy `ansible all -m ansible.builtin.ping -i inventory.yml` — kỳ vọng cả 3 host trả về `pong`.

**Kết quả cần có:** file `inventory.yml` hoàn chỉnh, output `ping` thành công cho cả 3 host.

## Lab 2 — Tạo `ansible.cfg` cho dự án

**Mục tiêu:** không phải gõ `-i inventory.yml` mỗi lần chạy lệnh.

**Các bước:**

1. Tạo file `ansible.cfg` trong cùng thư mục với `inventory.yml`, khai báo `inventory`, `remote_user`, `host_key_checking`, `forks`.
2. Chạy lại `ansible all -m ansible.builtin.ping` (không cần `-i`) — xác nhận vẫn hoạt động nhờ `ansible.cfg` được tự động đọc.
3. Chạy `ansible-config dump --only-changed` để xác nhận đúng giá trị bạn vừa khai báo đang được áp dụng.

**Kết quả cần có:** file `ansible.cfg`, output `ansible-config dump --only-changed` khớp với nội dung đã khai báo.

## Lab 3 — Ad-hoc Command cho tác vụ vận hành thật

**Mục tiêu:** luyện dùng đúng module chuyên biệt thay vì lạm dụng `command`/`shell`.

**Các bước:**

1. Cài `nginx` trên toàn bộ nhóm `webservers` bằng ad-hoc, dùng module quản lý package phù hợp hệ điều hành (`apt` hoặc `dnf`), có `-b`.
2. Kiểm tra service `nginx` đã chạy bằng module `service`/`systemd` (không dùng `command -a "systemctl status nginx"`).
3. Chạy lại đúng lệnh cài đặt ở bước 1 lần thứ hai, quan sát output — xác nhận `changed=0` (idempotent, không cài lại).
4. Copy một file `motd` tuỳ chọn tới `/etc/motd` trên toàn bộ host bằng module `copy`.

**Kết quả cần có:** log lần chạy đầu (`changed=...`) và lần chạy thứ hai (`changed=0`) đã lưu lại để so sánh.

## Lab 4 — Khám phá Facts

**Mục tiêu:** hiểu Facts khác Variables ở điểm nào, biết cách lọc facts cần thiết.

**Các bước:**

1. Chạy `ansible web-01 -m ansible.builtin.setup` — lưu toàn bộ output ra file, đếm số lượng khóa (key) cấp cao nhất trong JSON trả về.
2. Lọc riêng facts về mạng: `ansible web-01 -m ansible.builtin.setup -a "filter=ansible_default_ipv4"`.
3. Lọc riêng facts về hệ điều hành: `ansible web-01 -m ansible.builtin.setup -a "filter=ansible_distribution*"`.
4. So sánh facts giữa `web-01` và `db-01` nếu hai máy dùng hệ điều hành khác nhau (hoặc phiên bản khác nhau) — ghi lại điểm khác biệt.

**Kết quả cần có:** file log facts đầy đủ của `web-01`, và bảng so sánh ngắn 3-5 facts khác nhau giữa 2 host.

## Lab 5 — Viết Dynamic Inventory script tối giản

**Mục tiêu:** hiểu cơ chế dynamic inventory ở mức thực hành, không cần cloud thật.

**Các bước:**

1. Viết một script Python (`dynamic_inventory.py`, có quyền thực thi) trả về JSON đúng schema Ansible khi gọi với tham số `--list`, mô tả lại đúng 3 host ở Lab 1 (hard-code danh sách, mô phỏng việc "truy vấn" một nguồn dữ liệu ngoài).
2. Chạy `ansible-inventory -i dynamic_inventory.py --graph` — xác nhận cấu trúc nhóm hiển thị đúng như inventory tĩnh ở Lab 1.
3. Chạy `ansible all -m ansible.builtin.ping -i dynamic_inventory.py` — xác nhận vẫn kết nối được tới cả 3 host qua inventory động.

**Kết quả cần có:** file `dynamic_inventory.py` chạy được, output `--graph` và `ping` khớp kỳ vọng.

## Checklist hoàn thành module

- [ ] Cài đặt thành công Ansible trên control node, `ansible --version` chạy đúng.
- [ ] Viết được inventory tĩnh YAML với nhóm cha/con, `ansible-inventory --graph` hiển thị đúng cấu trúc.
- [ ] `ansible.cfg` hoạt động, không cần `-i` mỗi lần chạy lệnh.
- [ ] Chạy ad-hoc cài package + quản lý service + copy file, xác nhận idempotency qua `changed=0` ở lần chạy lặp lại.
- [ ] Thu thập và lọc được Facts, phân biệt rõ với Variables.
- [ ] Viết được một dynamic inventory script tối giản, chạy đúng qua `ansible-inventory`.
