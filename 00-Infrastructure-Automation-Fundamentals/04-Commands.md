---
title: "04 - Commands"
module: 0
tags: [ansible, sysops-infra, module-00, commands, idempotency, git]
---

# 04 — Lệnh & Ví dụ cấu hình mẫu

> Module này chưa cài Ansible (sẽ làm ở Module 02). Các lệnh dưới đây dùng công cụ bạn đã biết từ khóa Linux Sysadmin (`bash`, `ssh`, `git`) để **cảm nhận thực tế** các khái niệm ở [[02-Theory|02 - Theory]] trước khi chạm vào Ansible.

## 1. Kiểm tra điều kiện tiên quyết cho mô hình Agentless

Ansible (Module 02) chỉ cần hai thứ đã có sẵn trên managed node: SSH server đang chạy, và Python đã cài. Kiểm tra lại kiến thức Linux Sysadmin:

```bash
# Trên control node: kiểm tra ket noi SSH toi managed node (khong can gi them ngoai key da add)
ssh -o BatchMode=yes -o ConnectTimeout=5 deploy@192.168.56.11 'echo "SSH OK"'

# Tren managed node: kiem tra Python co san (Ansible can Python de chay module)
ssh deploy@192.168.56.11 'python3 --version'

# Kiem tra sshd dang chay tren managed node
ssh deploy@192.168.56.11 'systemctl is-active sshd'
```

Nếu cả ba lệnh trên chạy được không lỗi, managed node của bạn đã **sẵn sàng cho Ansible** — không cần cài thêm agent nào, đúng với mô hình Agentless đã học.

## 2. Demo Idempotency bằng shell — thấy sự khác biệt bằng mắt

### Ví dụ KHÔNG idempotent

```bash
# Chay lan 1: them dong vao /etc/hosts
echo "192.168.56.20 app.local" >> /etc/hosts

# Chay lan 2 (voi cung lenh nay): them THEM mot dong nua -> trung lap
echo "192.168.56.20 app.local" >> /etc/hosts

# Kiem tra: se thay 2 dong giong het nhau
grep "app.local" /etc/hosts
```

### Ví dụ CÓ idempotent (tự kiểm tra trước khi hành động)

```bash
# Chi them dong neu chua ton tai - chay bao nhieu lan cung chi co 1 dong
grep -qxF "192.168.56.20 app.local" /etc/hosts || echo "192.168.56.20 app.local" >> /etc/hosts
```

Chạy lệnh thứ hai bao nhiêu lần cũng chỉ cho đúng một kết quả: `/etc/hosts` có đúng một dòng `app.local`. Đây chính xác là nguyên lý mà module `ansible.builtin.lineinfile` (sẽ học ở Module 03/04) tự động hóa cho bạn — bạn sẽ không phải tự viết `grep -q || echo` nữa, nhưng cần hiểu **bên trong nó đang làm đúng logic này**.

### Ví dụ idempotent với thư mục

```bash
# KHONG idempotent: lan 2 se bao loi "File exists"
mkdir /opt/myapp

# Idempotent: chay bao nhieu lan cung khong loi, ket qua cuoi giong nhau
mkdir -p /opt/myapp
```

## 3. Git cơ bản cho một IaC repository

Trong doanh nghiệp, mọi playbook/inventory phải nằm trong Git — đây là nền tảng của GitOps đã học ở [[02-Theory|02 - Theory]] mục 7.

```bash
# Khoi tao repo IaC moi
mkdir infra-repo && cd infra-repo
git init

# Cau truc thu muc IaC dien hinh (se hoc chi tiet tu Module 02)
mkdir -p inventories playbooks roles group_vars host_vars

# .gitignore toi thieu cho mot repo Ansible - KHONG bao gio commit secret plaintext
cat > .gitignore <<EOG
*.retry
.vault_pass
*.pem
EOG

git add .
git commit -m "chore: khoi tao cau truc IaC repo"

# Workflow chuan: tao nhanh rieng cho moi thay doi ha tang, khong commit thang vao main
git checkout -b feature/add-web-server-config
# ... sua file playbook ...
git add .
git commit -m "feat: them cau hinh web server"
git push origin feature/add-web-server-config
# -> Mo Pull Request, cho review, roi moi merge (dung tinh than "Review" o Lifecycle)
```

## 4. Preview: các lệnh bạn sẽ dùng từ Module 02 trở đi

Chỉ để hình dung — **chưa cần chạy** ở module này:

```bash
ansible --version          # kiem tra Ansible da cai (Module 02)
ansible all -m ping        # ad-hoc command dau tien (Module 02)
ansible-playbook site.yml  # chay mot playbook (Module 03)
ansible-vault encrypt secrets.yml   # ma hoa secret (Module 05)
```
