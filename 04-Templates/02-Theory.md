---
title: "02 - Theory"
module: 4
tags: [ansible, sysops-infra, module-04, jinja2, filters, lookup, template]
---

# 02 — Lý thuyết

## 1. Jinja2 nâng cao — điều kiện và vòng lặp trong template

Ở Module 01, bạn học `{{ }}` (xuất giá trị) và nhận diện `{% %}` (điều khiển luồng). Trong file `.j2` thật, `{% %}` được dùng đầy đủ cho `if`/`for`:

```jinja2
{# nginx.conf.j2 #}
server {
    listen {{ http_port }};
    server_name {{ domain_name }};

{% if enable_ssl %}
    listen 443 ssl;
    ssl_certificate {{ ssl_cert_path }};
{% endif %}

    location / {
{% for upstream in upstream_servers %}
        # upstream: {{ upstream }}
{% endfor %}
        proxy_pass http://backend;
    }
}
```

Điểm khác biệt quan trọng so với template ngôn ngữ khác: `{% if %}`/`{% endif %}`, `{% for %}`/`{% endfor %}` **luôn phải đóng tường minh** — quên `{% endif %}`/`{% endfor %}` là lỗi cú pháp phổ biến nhất khi mới viết template dài.

## 2. Module `template` — khác biệt cốt lõi với `copy`

| Đặc điểm | `ansible.builtin.copy` | `ansible.builtin.template` |
|---|---|---|
| Xử lý cú pháp Jinja2 trong file | Không — chép nguyên văn | Có — render `{{ }}`/`{% %}` trước khi ghi |
| Đuôi file nguồn thường dùng | Bất kỳ | `.j2` (quy ước, không bắt buộc) |
| Dùng khi | File giống hệt nhau trên mọi host | File cần khác nhau theo host/biến |
| Idempotent | Có | Có — chỉ ghi lại nếu nội dung render ra khác file hiện tại |

```yaml
- name: Render cau hinh nginx rieng cho tung host
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/{{ domain_name }}.conf
    owner: root
    group: root
    mode: "0644"
  notify: Restart nginx
```

**Lưu ý quan trọng:** `template` tự động thêm một số tham số quản lý file như `owner`, `group`, `mode` giống `copy`/`file` — luôn khai báo tường minh `mode` thay vì để mặc định, tránh phụ thuộc vào umask không nhất quán giữa các managed node.

## 3. Filters — biến đổi giá trị ngay trong biểu thức

Filter dùng ký hiệu `|`, áp dụng một hàm biến đổi lên giá trị bên trái. Có thể nối nhiều filter liên tiếp.

| Filter | Chức năng | Ví dụ |
|---|---|---|
| `default` | Giá trị dự phòng nếu biến chưa định nghĩa | `{{ http_port \| default(80) }}` |
| `upper` / `lower` | Chuyển hoa/thường | `{{ env_name \| upper }}` |
| `join` | Nối list thành chuỗi | `{{ packages \| join(', ') }}` |
| `to_json` / `to_yaml` | Chuyển sang chuỗi JSON/YAML | `{{ my_dict \| to_json }}` |
| `regex_replace` | Thay thế theo regex | `{{ domain \| regex_replace('\.local$', '') }}` |
| `int` / `bool` | Ép kiểu dữ liệu | `{{ "8080" \| int }}` |
| `unique` | Loại phần tử trùng trong list | `{{ my_list \| unique }}` |
| `map('attribute', 'x')` | Trích một thuộc tính từ list dictionary | `{{ users \| map(attribute='name') \| list }}` |

Filter `default` đặc biệt quan trọng trong thực tế: giúp Template/Playbook **không lỗi crash** khi một biến chưa được khai báo cho một host cụ thể, thay vào đó dùng giá trị an toàn mặc định — kỹ thuật phòng thủ (defensive) tiêu chuẩn khi viết template dùng chung cho nhiều môi trường.

## 4. Lookup Plugin — lấy dữ liệu từ ngoài Ansible

Lookup cho phép đưa dữ liệu từ **nguồn bên ngoài** (không phải inventory/vars có sẵn) vào biến, thực thi trên control node.

```yaml
vars:
  # Doc noi dung mot file cuc bo tren control node
  ssh_pubkey: "{{ lookup('ansible.builtin.file', '~/.ssh/id_automation.pub') }}"

  # Doc mot bien moi truong cua control node
  build_number: "{{ lookup('ansible.builtin.env', 'BUILD_NUMBER') }}"

  # Chay mot lenh shell tren control node, lay output
  git_commit: "{{ lookup('ansible.builtin.pipe', 'git rev-parse --short HEAD') }}"
```

**Phân biệt quan trọng:** Lookup chạy trên **Control Node** (nơi Ansible được gọi), khác với Facts (thu thập từ Managed Node) và Task thông thường (thực thi trên Managed Node). Dùng sai chỗ — ví dụ tưởng `lookup('file', ...)` đọc file trên managed node — là lỗi hiểu sai phổ biến của người mới.

## 5. Bốn module xử lý file — chọn đúng công cụ

| Module | Chiều dữ liệu | Chức năng chính |
|---|---|---|
| `ansible.builtin.file` | Không di chuyển dữ liệu | Quản lý thuộc tính: quyền, chủ sở hữu, tạo thư mục/symlink, xóa file |
| `ansible.builtin.copy` | Control Node → Managed Node | Chép file/nội dung nguyên văn, không render Jinja2 |
| `ansible.builtin.template` | Control Node → Managed Node | Render Jinja2 rồi chép, dùng cho file cấu hình động |
| `ansible.builtin.fetch` | Managed Node → Control Node | **Chiều ngược lại** — lấy file từ managed node về (ví dụ log, báo cáo) |
| `ansible.posix.synchronize` | Control Node ↔ Managed Node | Đồng bộ cả thư mục qua rsync, hiệu quả cho lượng dữ liệu lớn |

```yaml
- name: Tao thu muc voi quyen cu the
  ansible.builtin.file:
    path: /var/www/myapp
    state: directory
    owner: www-data
    mode: "0755"

- name: Lay log loi ve control node de phan tich
  ansible.builtin.fetch:
    src: /var/log/nginx/error.log
    dest: ./logs/{{ inventory_hostname }}-error.log
    flat: true

- name: Dong bo toan bo thu muc ma nguon (nhanh hon copy voi nhieu file)
  ansible.posix.synchronize:
    src: ./dist/
    dest: /var/www/myapp/
    delete: true
```

**Vì sao `fetch` quan trọng trong vận hành thực tế:** đây là cách chuẩn để thu thập log/artifact từ hàng chục server về một chỗ để phân tích tập trung, thay vì SSH thủ công vào từng máy — `fetch` tự động thêm tên host vào đường dẫn đích (mặc định) để tránh ghi đè file cùng tên từ các host khác nhau.

**Vì sao `synchronize` nhanh hơn `copy` với nhiều file:** `synchronize` dùng `rsync` bên dưới, chỉ truyền phần dữ liệu thay đổi (delta) thay vì toàn bộ file mỗi lần — hiệu quả rõ rệt khi triển khai thư mục mã nguồn lớn (ví dụ build artifact frontend) lên nhiều server.
