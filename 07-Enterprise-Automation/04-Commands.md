---
title: "04 - Commands"
module: 7
tags: [ansible, sysops-infra, module-07, commands, serial, haproxy]
---

# 04 — Lệnh & Cú pháp tham chiếu

## 1. `serial` — các dạng khai báo

```yaml
# So co dinh
serial: 2

# Phan tram tren tong so host cua Play
serial: "25%"

# Progressive / canary - tang dan qua tung dot
serial:
  - 1
  - 5
  - "25%"
  - "100%"
```

## 2. `max_fail_percentage` kết hợp `serial`

```yaml
- hosts: webservers
  serial: 4
  max_fail_percentage: 20
  tasks:
    - name: Cap nhat ung dung
      ansible.builtin.git:
        repo: "https://git.example.com/app.git"
        dest: /opt/myapp
```

## 3. `delegate_to` — chạy Task trên host khác

```yaml
- name: Cau hinh lai HAProxy tu mot web server
  ansible.builtin.template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
  delegate_to: "{{ groups['loadbalancers'][0] }}"
  run_once: true
```

## 4. Retry/Health check với `until`

```yaml
- name: Cho ung dung san sang truoc khi tiep tuc
  ansible.builtin.uri:
    url: "http://localhost/healthz"
    status_code: 200
  register: health
  until: health.status == 200
  retries: 10
  delay: 5
```

## 5. Biến "magic" `groups` và `hostvars`

```jinja2
{# Liet ke toan bo host trong 1 nhom #}
{% for host in groups['webservers'] %}
  {{ host }}
{% endfor %}

{# Lay bien/facts cua mot host KHAC, khong phai host hien tai #}
{{ hostvars['web-01']['ansible_default_ipv4']['address'] }}
```

## 6. `run_once` — chỉ chạy một lần cho toàn Play

```yaml
- name: Gui thong bao trien khai hoan tat mot lan duy nhat
  ansible.builtin.uri:
    url: "https://monitoring.example.com/api/events"
    method: POST
  delegate_to: localhost
  run_once: true
```

## 7. Chạy thử Rolling Update an toàn trước production

```bash
# Xem truoc so dot, so host moi dot se chay nhu the nao
ansible-playbook rolling-update.yml --list-hosts

# Dry run toan bo, khong thay doi gi
ansible-playbook rolling-update.yml --check --diff

# Gioi han chi chay tren 1 dot dau tien de kiem chung an toan (canary)
ansible-playbook rolling-update.yml --limit "web-01,web-02"
```

## 8. Bảng tổng hợp tham số kiểm soát Rolling Update

| Tham số | Chức năng |
|---|---|
| `serial` | Số/phần trăm host xử lý mỗi đợt |
| `max_fail_percentage` | Ngưỡng % thất bại để dừng toàn bộ Playbook |
| `delegate_to` | Chạy Task trên host khác (ví dụ Load Balancer) |
| `run_once` | Chỉ chạy Task một lần cho toàn Play |
| `until`/`retries`/`delay` | Retry cho tới khi điều kiện đúng (health check) |
| `groups` / `hostvars` | Truy cập thông tin host khác trong cùng Playbook |
