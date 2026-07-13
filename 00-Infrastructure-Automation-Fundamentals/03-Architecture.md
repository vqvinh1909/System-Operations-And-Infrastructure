---
title: "03 - Architecture"
module: 0
tags: [ansible, sysops-infra, module-00, architecture, diagram]
---

# 03 — Kiến trúc & Sơ đồ

## Sơ đồ 1: Infrastructure Lifecycle (vòng đời hạ tầng)

```
+-------------------------------------+
|               1. PLAN               |
|           (Lap ke hoach)            |
+-------------------------------------+
                   |
                   v
+-------------------------------------+
|        2. WRITE / PROVISION         |
|           (Viet code IaC)           |
+-------------------------------------+
                   |
                   v
+-------------------------------------+
|              3. REVIEW              |
|           (Pull Request)            |
+-------------------------------------+
                   |
                   v
+-------------------------------------+
|              4. APPLY               |
|     (Ap dung len ha tang that)      |
+-------------------------------------+
                   |
                   v
+-------------------------------------+
|         5. TEST / VALIDATE          |
|         (Kiem tra ket qua)          |
+-------------------------------------+
                   |
                   v
+-------------------------------------+
|             6. MONITOR              |
|         (Giam sat lien tuc)         |
+-------------------------------------+
                   |
                   v
+-------------------------------------+
|              7. UPDATE              |
|          (Co yeu cau moi)           |
+-------------------------------------+
                   |
                   +---> quay lai buoc 1. PLAN (vong lap lien tuc)
```

Đọc sơ đồ: đây **không phải** một quy trình chạy một lần rồi kết thúc — nó là một **vòng lặp liên tục**. Mỗi khi có yêu cầu mới (patch bảo mật, thêm server, đổi cấu hình), bạn quay lại bước 1 và đi hết vòng lặp. Bước 3 (Review) là bước phân biệt rõ nhất IaC với cách làm thủ công — không có bước tương đương nào trong mô hình Traditional ở [[02-Theory|02 - Theory]].

## Sơ đồ 2: Kiến trúc Agentless (Ansible)

```
+----------------------------------+
|           CONTROL NODE           |
|   - Cai Ansible (pip/dnf/apt)    |
|    - Luu inventory, playbook     |
|        - Co SSH key trong        |
|     authorized_keys cua target   |
+----------------------------------+
                  |
          SSH (port 22, dung ansible_user + key)
                  v
+----------------------------------+
|   PUSH module Python qua SSH,    |
|      thuc thi ngay lap tuc,      |
|   roi TU XOA - khong de lai gi   |
+----------------------------------+
                  |
                  v
+----------------------------------+
|           MANAGED NODE           |
|     - Chi can: Python co san     |
|         - sshd dang chay         |
|  - KHONG can cai them agent nao  |
+----------------------------------+
```

Đọc sơ đồ: đây là điểm khác biệt cốt lõi giữa Ansible (agentless, push model) và Puppet/Chef (agent-based, pull model). Ansible **tận dụng lại đúng kênh SSH** mà bạn đã hardening ở khóa Linux Sysadmin (Module 20 — SSH Hardening) — không mở thêm cổng, không cài thêm phần mềm thường trực nào trên managed node. Module Python được đẩy (push) sang, chạy, rồi bị xóa — không để lại tiến trình nào chạy nền.

## Sơ đồ 3: Enterprise IaC Architecture (kiến trúc cấp doanh nghiệp)

```
+------------------------------+
|    Ky su (Dev / Sysadmin)    |
+------------------------------+
                |
        git commit + push
                v
+------------------------------+
|        GIT REPOSITORY        |
| (Playbook, Role, Inventory)  |
+------------------------------+
                |
    Pull Request -> Review -> Merge
                v
+------------------------------+
|        CI/CD PIPELINE        |
| (ansible-lint, syntax check, |
|       --check dry-run)       |
+------------------------------+
                |
        trigger deploy
                v
+------------------------------+
|         CONTROL NODE         |
| (ansible-playbook, doc Vault |
|   de lay secrets khi can)    |
+------------------------------+
                |
          +----------+----------+
          |          |          |
          v          v          v
        +------------+   +------------+   +------------+
        | Web Server |   | DB Server  |   | Cache / LB |
        +------------+   +------------+   +------------+
```

Đọc sơ đồ: đây là bức tranh tổng thể mà Module 07 (cuối Part I) sẽ hiện thực hóa bằng playbook thật. Điểm mấu chốt: **không ai chạy `ansible-playbook` trực tiếp từ laptop lên production** trong môi trường doanh nghiệp trưởng thành — mọi thay đổi đi qua Git, được review, được CI/CD kiểm tra tự động, rồi mới chạm tới Control Node. Đây chính là tinh thần GitOps đã giới thiệu ở [[02-Theory|02 - Theory]] mục 7.
