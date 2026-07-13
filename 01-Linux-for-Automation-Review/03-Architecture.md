---
title: "03 - Architecture"
module: 1
tags: [ansible, sysops-infra, module-01, architecture, diagram]
---

# 03 — Kiến trúc & Sơ đồ

## Sơ đồ 1: Luồng xác thực SSH key khi Ansible kết nối

```
+-----------------------------------------+
|              CONTROL NODE               |
| - co ~/.ssh/id_automation (private key) |
|    - ~/.ssh/config: alias, user, key    |
+-----------------------------------------+
                    |
      1. Ket noi SSH toi managed node
                    v
+-----------------------------------+
|           MANAGED NODE            |
| - authorized_keys chua PUBLIC KEY |
|  tuong ung voi id_automation.pub  |
+-----------------------------------+
                    |
      2. sshd so khop public/private key
                    v
+-----------------------------------------+
| Khop: cap phien SSH, KHONG hoi password |
+-----------------------------------------+
                    |
      3. Ansible push module Python qua phien SSH
                    v
+-----------------------------------+
| Module thuc thi, tra ket qua JSON |
|        ve lai control node        |
+-----------------------------------+
```

Đọc sơ đồ: đây là toàn bộ "phép màu" mà Module 02 sẽ dựa vào khi bạn chạy `ansible all -m ping` lần đầu tiên — thực chất không có gì huyền bí, chỉ là SSH key-based authentication bạn đã học ở khóa Linux Sysadmin, cộng thêm bước push và thực thi module Python. Nếu bước 1-2 thất bại (sai key, `authorized_keys` chưa đúng), Ansible sẽ báo lỗi `UNREACHABLE` — lỗi đầu tiên và phổ biến nhất người mới học Ansible gặp phải, xử lý ở [[06-Troubleshooting|06 - Troubleshooting]].

## Sơ đồ 2: Một host thuộc nhiều nhóm inventory cùng lúc

```
                    +--------------------+
                    |       web-01       |
                    +--------------------+
                              |
                +-------------+-------------+
                |             |             |
                v             v             v
        +--------------+ +--------------+ +--------------+
        |  production  | |  webservers  | |   dc-hanoi   |
        +--------------+ +--------------+ +--------------+
```

Đọc sơ đồ: `web-01` đồng thời thuộc 3 nhóm — theo môi trường (`production`), theo vai trò (`webservers`), theo vị trí (`dc-hanoi`). Đây là tư duy phân nhóm bạn cần lập kế hoạch **trước khi** viết inventory Ansible thật ở Module 02. Lợi ích thực tế: bạn có thể áp dụng một thay đổi cho toàn bộ `production` (bất kể vai trò), hoặc chỉ cho `webservers` (bất kể môi trường), hoặc giao cả hai điều kiện cùng lúc — sự linh hoạt này chỉ có được nếu bạn thiết kế nhóm hợp lý ngay từ đầu, thay vì nhét mọi host vào một danh sách phẳng.
