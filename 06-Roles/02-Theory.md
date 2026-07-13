---
title: "02 - Theory"
module: 6
tags: [ansible, sysops-infra, module-06, roles, galaxy, collections, dependencies]
---

# 02 — Lý thuyết

## 1. Cấu trúc thư mục Role chuẩn

```
roles/nginx_hardened/
├── tasks/
│   └── main.yml          # Danh sach Task chinh, diem vao cua Role
├── handlers/
│   └── main.yml          # Handler dung rieng cho Role nay
├── templates/
│   └── nginx.conf.j2     # File Jinja2 template
├── files/
│   └── robots.txt        # File tinh, chep nguyen van
├── vars/
│   └── main.yml          # Bien noi bo, uu tien CAO, khong danh cho nguoi dung sua
├── defaults/
│   └── main.yml          # Bien mac dinh, uu tien THAP NHAT, danh cho nguoi dung ghi de
├── meta/
│   └── main.yml          # Metadata: dependencies, thong tin tac gia, platform ho tro
└── README.md              # Tai lieu: Role lam gi, bien nao co the cau hinh
```

Ansible **tự động nhận diện** cấu trúc này — chỉ cần đặt đúng tên thư mục con, không cần khai báo thủ công "file tasks nằm ở đâu". Đây là lý do mọi kỹ sư Ansible trên thế giới đọc được Role của nhau ngay lập tức, dù chưa từng làm việc chung — sức mạnh của một **quy ước chuẩn hóa** (convention over configuration).

## 2. `tasks/main.yml` — điểm vào của Role

```yaml
---
- name: Cai dat nginx
  ansible.builtin.apt:
    name: nginx
    state: present

- name: Trien khai cau hinh
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx
```

Mọi Task, cú pháp (loop, when, block, tags — Module 03) hoạt động **y hệt** bên trong Role như trong Playbook thường — Role không phải cú pháp mới, chỉ là cách tổ chức vị trí file.

## 3. `defaults/main.yml` vs `vars/main.yml` — khác biệt cốt lõi

Đây là điểm gây nhầm lẫn nhiều nhất với người mới học Role — cả hai đều "chứa biến", nhưng độ ưu tiên (đã học ở Module 05) và mục đích hoàn toàn khác nhau:

| Đặc điểm | `defaults/main.yml` | `vars/main.yml` |
|---|---|---|
| Độ ưu tiên (Module 05) | **Thấp nhất** trong toàn hệ thống (cấp 2) | Cao (cấp 15) |
| Mục đích | Giá trị mặc định **có thể/nên** bị người dùng Role ghi đè | Giá trị nội bộ, **không nên** bị ghi đè tùy tiện |
| Ví dụ phù hợp | `nginx_worker_connections: 1024` (người dùng có thể muốn 2048) | `nginx_config_path: /etc/nginx/nginx.conf` (đường dẫn cố định, không nên đổi) |

```yaml
# defaults/main.yml - nguoi dung Role de dang ghi de
nginx_worker_connections: 1024
nginx_port: 80

# vars/main.yml - gia tri noi bo, Role tu quyet dinh
nginx_config_path: /etc/nginx/nginx.conf
nginx_service_name: nginx
```

**Nguyên tắc thiết kế Role tốt:** mọi giá trị bạn **kỳ vọng** người dùng Role sẽ muốn tùy chỉnh (port, số lượng worker, tên miền) phải nằm ở `defaults`. Giá trị bạn **không muốn** ai vô tình ghi đè (đường dẫn file cấu hình cố định theo hệ điều hành, tên service cố định) nên nằm ở `vars`.

## 4. `meta/main.yml` — Metadata và Dependencies

```yaml
---
galaxy_info:
  author: Vinh Sysadmin
  description: Cai dat va hardening nginx theo chuan cong ty
  license: MIT
  min_ansible_version: "2.19"
  platforms:
    - name: Ubuntu
      versions: [jammy, noble]
    - name: EL
      versions: ["9"]

dependencies:
  - role: common_hardening
  - role: firewall_setup
    vars:
      firewall_allowed_ports: [80, 443]
```

`dependencies` khai báo Role này **phụ thuộc** Role khác — Ansible tự động chạy các Role phụ thuộc **trước**, theo thứ tự khai báo, trước khi chạy Role hiện tại. Ví dụ trên: `common_hardening` và `firewall_setup` luôn chạy trước `nginx_hardened` mỗi khi Role này được gọi, không cần Playbook chính khai báo lại thủ công.

> Lưu ý thực tế quan trọng: theo mặc định, một Role dependency **chỉ chạy một lần** nếu được nhiều Role khác nhau cùng phụ thuộc trong cùng một Playbook (Ansible tự loại trùng dựa trên tên Role + tham số giống hệt nhau) — tránh chạy lặp lại không cần thiết. Có thể ép chạy lại bằng tham số `allow_duplicates: true` khi thực sự cần.

## 5. Ansible Galaxy — kho Role/Collection cộng đồng

**Ansible Galaxy** (`galaxy.ansible.com`) là kho lưu trữ công khai nơi cộng đồng chia sẻ Role và Collection. Thay vì tự viết lại "cấu hình PostgreSQL chuẩn" từ đầu, có thể tìm Role/Collection đã được kiểm chứng rộng rãi:

```bash
# Tim kiem tren galaxy (qua giao dien web hoac CLI)
ansible-galaxy search postgresql

# Cai mot role tu Galaxy
ansible-galaxy role install geerlingguy.postgresql

# Cai mot collection tu Galaxy
ansible-galaxy collection install community.postgresql
```

**Nguyên tắc dùng Role/Collection bên thứ ba trong doanh nghiệp:** luôn review mã nguồn trước khi dùng cho production (không chỉ tin số lượt tải), ghim (pin) đúng version trong `requirements.yml` thay vì luôn lấy bản mới nhất (tránh thay đổi hành vi bất ngờ giữa các lần chạy CI/CD), và ưu tiên Role/Collection có tác giả/tổ chức uy tín, còn được bảo trì tích cực.

## 6. Collections — đơn vị đóng gói từ Ansible 2.10 trở đi

**Collection** là cách đóng gói chính thức, hiện đại hơn Role đơn lẻ — một Collection có thể chứa **nhiều Role, Module, Plugin, Lookup, Filter** cùng lúc, đặt tên theo namespace `<tổ_chức>.<tên_collection>` (ví dụ `amazon.aws`, `community.general`, `ansible.posix` đã gặp ở Module 04).

```yaml
# requirements.yml - khai bao dependencies chuan hoa cho ca role va collection
---
roles:
  - name: geerlingguy.postgresql
    version: "3.5.0"

collections:
  - name: community.general
    version: ">=8.0.0"
  - name: amazon.aws
```

```bash
# Cai toan bo dependencies khai bao trong requirements.yml mot lenh
ansible-galaxy install -r requirements.yml
```

**Phân biệt nhanh:** Role là "một chức năng đóng gói sẵn" (ví dụ "cài PostgreSQL"). Collection là "một bộ công cụ đóng gói sẵn" (ví dụ toàn bộ module/plugin để làm việc với AWS) — có thể **chứa** Role bên trong nhưng phạm vi rộng hơn nhiều. Kể từ khi mô hình Collection ra đời, hầu hết module built-in trước đây (`ansible.builtin.*`) và module cộng đồng đều được phân phối dưới dạng Collection, dù `ansible.builtin` luôn có sẵn không cần cài thêm.

## 7. Best Practices khi thiết kế Role

- **Một Role làm đúng một việc** (single responsibility) — tránh Role "làm tất cả" khó tái sử dụng và khó test.
- **Luôn viết `README.md`** trong Role: mô tả chức năng, liệt kê biến có thể cấu hình (từ `defaults`) kèm giá trị mặc định và ý nghĩa.
- **Đặt tên biến có tiền tố tên Role** (ví dụ `nginx_worker_connections` thay vì chỉ `worker_connections`) — tránh xung đột tên biến khi nhiều Role cùng chạy trong một Playbook.
- **Idempotent xuyên suốt** — Role chạy lại nhiều lần phải cho kết quả nhất quán, đúng nguyên lý xuyên suốt khóa học từ Module 00.
- **Test Role độc lập** trước khi ráp vào Playbook lớn — dùng `ansible-playbook` với một Playbook tối giản chỉ gọi đúng Role đó để cô lập lỗi.
