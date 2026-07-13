---
title: "04 - Commands"
module: 4
tags: [ansible, sysops-infra, module-04, commands, template, jinja2]
---

# 04 — Lệnh & Cú pháp tham chiếu

## 1. Module `template` — cú pháp đầy đủ

```yaml
- name: Render file cau hinh dong
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
    owner: root
    group: root
    mode: "0644"
    backup: true
    validate: "nginx -t -c %s"
```

`backup: true` tự động tạo bản sao lưu (timestamp) của file cũ trước khi ghi đè — hữu ích để rollback thủ công nếu cần. `validate` chạy một lệnh kiểm tra cú pháp file **trước khi** áp dụng thật (`%s` là placeholder Ansible tự thay bằng đường dẫn file tạm) — nếu lệnh validate thất bại, Ansible không ghi đè file gốc, tránh đẩy một file cấu hình sai vào production.

## 2. Kiểm tra template trước khi triển khai thật

```bash
# Xem truoc noi dung se duoc render, khong ghi that
ansible-playbook site.yml --check --diff --tags config
```

`--diff` hiển thị dạng unified diff giữa file cũ và nội dung sẽ render — cách nhanh nhất để review thay đổi cấu hình trước khi áp dụng lên production.

## 3. Filters — cú pháp mẫu

```jinja2
{{ http_port | default(80) }}
{{ env_name | upper }}
{{ packages | join(', ') }}
{{ my_dict | to_json }}
{{ my_dict | to_nice_yaml }}
{{ "8080" | int }}
{{ users | map(attribute='name') | list }}
{{ domain | regex_replace('\.local$', '') }}
```

## 4. Lookup — cú pháp mẫu

```yaml
vars:
  pubkey_content: "{{ lookup('ansible.builtin.file', '~/.ssh/id_automation.pub') }}"
  home_dir: "{{ lookup('ansible.builtin.env', 'HOME') }}"
  commit_hash: "{{ lookup('ansible.builtin.pipe', 'git rev-parse --short HEAD') }}"
  random_pw: "{{ lookup('ansible.builtin.password', '/dev/null length=16') }}"
```

## 5. Bốn module xử lý file — cú pháp mẫu

```yaml
# file - quan ly thuoc tinh
- name: Tao thu muc
  ansible.builtin.file:
    path: /var/www/myapp
    state: directory
    owner: www-data
    group: www-data
    mode: "0755"

# copy - chep tinh
- name: Chep file tinh nguyen van
  ansible.builtin.copy:
    src: files/robots.txt
    dest: /var/www/myapp/robots.txt

# fetch - lay ve tu managed node
- name: Lay log ve control node
  ansible.builtin.fetch:
    src: /var/log/nginx/access.log
    dest: ./collected-logs/
    flat: false

# synchronize - dong bo thu muc (can cai package rsync tren ca 2 phia)
- name: Dong bo thu muc build
  ansible.posix.synchronize:
    src: ./dist/
    dest: /var/www/myapp/
    delete: true
    rsync_opts:
      - "--exclude=.git"
```

## 6. Bảng tổng hợp Filter thường dùng nhất

| Filter | Chức năng |
|---|---|
| `default(val)` | Giá trị dự phòng nếu biến undefined |
| `upper` / `lower` | Đổi hoa/thường |
| `join(sep)` | Nối list thành chuỗi |
| `length` | Đếm số phần tử |
| `to_json` / `from_json` | Chuyển đổi qua lại JSON |
| `to_nice_yaml` | Xuất YAML dễ đọc |
| `int` / `float` / `bool` | Ép kiểu |
| `regex_replace(a, b)` | Thay thế theo regex |
| `unique` | Loại phần tử trùng |
| `map(attribute='x')` | Trích thuộc tính từ list dictionary |
| `selectattr('x', 'equalto', v)` | Lọc list dictionary theo điều kiện |
