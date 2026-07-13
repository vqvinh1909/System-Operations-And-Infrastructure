---
title: "01 - Introduction"
module: 6
tags: [ansible, sysops-infra, module-06, introduction]
---

# 01 — Giới thiệu

## Bài toán: 5 dự án, cùng một đoạn Task "cài nginx chuẩn công ty"

Công ty bạn có 5 team, mỗi team quản lý một dự án Ansible riêng, nhưng tất cả đều cần "cài nginx theo chuẩn bảo mật công ty" — đúng phiên bản, đúng cấu hình hardening, đúng logging. Không có Role, mỗi team copy-paste cùng một chuỗi Task vào Playbook của mình. Sáu tháng sau, đội bảo mật phát hiện lỗ hổng, cần cập nhật cấu hình — giờ phải sửa **5 nơi khác nhau**, và không có gì đảm bảo cả 5 team đều đang chạy cùng một phiên bản logic (một team có thể đã tự sửa thêm, một team quên chưa cập nhật). Đây chính xác là kiểu nợ kỹ thuật Role được sinh ra để loại bỏ.

## Role là gì

Role là một **cấu trúc thư mục chuẩn hóa**, đóng gói toàn bộ Task, Handler, Template, File, Variables (kể cả Vault) liên quan tới một chức năng cụ thể (ví dụ "cấu hình nginx", "cấu hình PostgreSQL", "hardening bảo mật cơ bản") thành **một đơn vị độc lập, tái sử dụng được** giữa nhiều Playbook, nhiều dự án, thậm chí chia sẻ công khai qua Ansible Galaxy.

```yaml
# site.yml - Playbook chinh tro nen gon gang the nay khi dung Role
- hosts: webservers
  roles:
    - role: nginx_hardened
      vars:
        nginx_worker_connections: 2048
    - role: monitoring_agent
```

Toàn bộ logic chi tiết "cài nginx như thế nào" nằm gọn trong thư mục `roles/nginx_hardened/`, không còn nằm rải rác trong `site.yml`.

## Vì sao đây là module cuối cùng trước Enterprise Automation

Role là lớp trừu tượng hóa cao nhất trong Ansible core (trên Role chỉ còn Collection — đóng gói nhiều Role/Plugin/Module cùng lúc, học ở phần cuối module này). Mọi kỹ năng từ Module 01 tới 05 (SSH/YAML, Inventory/Module, Playbook/Task/Handler, Template/Jinja2, Variables/Vault) đều được **đóng gói lại** bên trong Role — đây là lý do Module 07 (Enterprise Automation) hầu như chỉ còn thao tác ở cấp độ "gọi Role nào, theo thứ tự nào" thay vì viết lại Task từ đầu.

## Mục tiêu học tập

Sau module này, bạn chuyển đổi được một Playbook đơn lẻ thành Role có cấu trúc chuẩn, hiểu rõ sự khác biệt `defaults` và `vars`, khai báo được Dependencies giữa các Role, và biết cách tìm/cài Role hoặc Collection chất lượng từ Ansible Galaxy thay vì luôn phải viết lại từ đầu.
