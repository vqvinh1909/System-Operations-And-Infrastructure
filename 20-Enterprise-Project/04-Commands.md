---
title: "04 - Commands"
module: 20
tags: [docker, ansible, sysops-infra, module-20, capstone, commands]
---

# 04 — Bộ lệnh tham khảo nhanh theo từng giai đoạn

> [!note] Đây là tài liệu tra cứu nhanh, không phải hướng dẫn từng bước
> Các lệnh dưới đây là bộ khung tham khảo cho từng giai đoạn của dự án. Chi tiết yêu cầu triển khai đầy đủ nằm ở [[05-Labs]] — bạn cần tự quyết định thứ tự và cách kết hợp các lệnh này cho đúng kiến trúc ở [[03-Architecture]].

## Giai đoạn 1: SSH Key và Inventory

```bash
# Tren Control Node - tao SSH key rieng cho Ansible (khong dung key ca nhan)
ssh-keygen -t ed25519 -f ~/.ssh/ansible_project -C "ansible-control-node"

# Copy public key sang tung managed node
ssh-copy-id -i ~/.ssh/ansible_project.pub deploy@192.168.10.20   # haproxy
ssh-copy-id -i ~/.ssh/ansible_project.pub deploy@192.168.10.31   # web01
ssh-copy-id -i ~/.ssh/ansible_project.pub deploy@192.168.10.32   # web02
ssh-copy-id -i ~/.ssh/ansible_project.pub deploy@192.168.10.40   # database

# Kiem tra ket noi khong can mat khau
ansible all -i inventory.ini -m ping
```

```ini
# inventory.ini
[haproxy]
haproxy01 ansible_host=192.168.10.20

[web]
web01 ansible_host=192.168.10.31
web02 ansible_host=192.168.10.32

[database]
db01 ansible_host=192.168.10.40

[all:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/ansible_project
```

## Giai đoạn 2: Cấu trúc Roles và Playbook

```bash
# Cau truc thu muc chuan (da hoc o Module 06 - Roles)
ansible-galaxy init roles/common
ansible-galaxy init roles/docker
ansible-galaxy init roles/haproxy
ansible-galaxy init roles/webapp
ansible-galaxy init roles/database
```

```yaml
# site.yml - playbook chinh, goi dung role theo dung group trong inventory
- hosts: all
  roles:
    - common
    - docker

- hosts: haproxy
  roles:
    - haproxy

- hosts: web
  roles:
    - webapp

- hosts: database
  roles:
    - database
```

```bash
# Chay toan bo
ansible-playbook -i inventory.ini site.yml

# Chay thu (khong ap dung that) de kiem tra truoc
ansible-playbook -i inventory.ini site.yml --check --diff

# Chay lai chi 1 nhom (huu ich khi debug rieng 1 tang)
ansible-playbook -i inventory.ini site.yml --limit web
```

## Giai đoạn 3: HAProxy (Template Jinja2)

```jinja2
{# roles/haproxy/templates/haproxy.cfg.j2 #}
frontend http_front
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health
{% for host in groups['web'] %}
    server {{ host }} {{ hostvars[host].ansible_host }}:8080 check
{% endfor %}
```

## Giai đoạn 4: Docker Compose theo từng vai trò

```yaml
# roles/webapp/files/docker-compose.yml (tren Web01/Web02)
services:
  nginx:
    image: nginx:1.27
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    restart: unless-stopped
```

```yaml
# roles/database/files/docker-compose.yml (tren Database)
services:
  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
    volumes:
      - mysql-data:/var/lib/mysql
    secrets:
      - mysql_root_password
    restart: unless-stopped

  redis:
    image: redis:7
    volumes:
      - redis-data:/data
    restart: unless-stopped

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"   # UI quan tri - CHI mo noi bo, khong public ra internet
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    restart: unless-stopped

secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt

volumes:
  mysql-data:
  redis-data:
  rabbitmq-data:
```

> [!note] Vì sao dùng `secrets` thay vì `environment` cho mật khẩu MySQL
> Đây là kiến thức đã học ở Module 05 (Variables, Vault, Secrets) — áp dụng lại: khai báo mật khẩu trực tiếp trong `environment` của Compose khiến giá trị dễ bị lộ qua `docker inspect`, log, hoặc file compose commit nhầm lên git. `secrets` (đọc từ file, kết hợp Ansible Vault để mã hóa file đó khi lưu trong repo) là thực hành an toàn hơn.

## Giai đoạn 5: Monitoring tập trung nhiều host

```yaml
# prometheus.yml tren VM giam sat (co the dat tren Control Node)
scrape_configs:
  - job_name: "cadvisor-fleet"
    static_configs:
      - targets:
          - "192.168.10.20:8080"   # haproxy
          - "192.168.10.31:8080"   # web01
          - "192.168.10.32:8080"   # web02
          - "192.168.10.40:8080"   # database
```

## Giai đoạn 6: Backup

```bash
# rsync - dong bo file cau hinh/ma nguon tinh, chay dinh ky bang cron
rsync -avz --delete \
  deploy@192.168.10.31:/opt/webapp/html/ \
  /backup/web01-html/

# Volume backup - dong goi toan bo du lieu MySQL tai 1 thoi diem
docker run --rm \
  -v database_mysql-data:/data:ro \
  -v /backup/mysql:/backup \
  alpine tar czf /backup/mysql-$(date +%F-%H%M).tar.gz -C /data .

# Khoi phuc tu ban backup volume
docker run --rm \
  -v database_mysql-data:/data \
  -v /backup/mysql:/backup \
  alpine sh -c "cd /data && tar xzf /backup/mysql-2026-07-10-0200.tar.gz"
```
