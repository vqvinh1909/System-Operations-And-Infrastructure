---
title: "04 - Commands"
module: 3
tags: [ansible, sysops-infra, module-03, commands, ansible-playbook]
---

# 04 — Lệnh & Cú pháp tham chiếu

## 1. Chạy Playbook cơ bản

```bash
# Chay playbook, dung inventory da khai bao trong ansible.cfg
ansible-playbook site.yml

# Chi dinh inventory tuong minh
ansible-playbook -i inventory.yml site.yml

# Chi dinh nhieu bien qua dong lenh
ansible-playbook site.yml --extra-vars "http_port=8080 env=staging"
```

## 2. Kiểm tra trước khi chạy thật (Dry Run)

```bash
# Check mode: mo phong, KHONG thay doi gi tren managed node
ansible-playbook site.yml --check

# Ket hop --check voi --diff de xem chi tiet thay doi du kien
ansible-playbook site.yml --check --diff
```

`--check` là công cụ quan trọng nhất trước khi chạy Playbook trên production lần đầu — cho biết Task nào sẽ báo `changed` mà không thực sự thực thi. Lưu ý: không phải module nào cũng hỗ trợ đầy đủ `--check` (một số module báo `changed` không chính xác trong chế độ này khi kết quả phụ thuộc trạng thái runtime phức tạp) — luôn đọc kỹ output, không tin tuyệt đối.

## 3. Giới hạn phạm vi chạy

```bash
# Chi chay tren mot host/nhom cu the, du Playbook khai bao hosts: all
ansible-playbook site.yml --limit webservers
ansible-playbook site.yml --limit web-01

# Chi chay Task co tag cu the
ansible-playbook site.yml --tags config

# Chay moi thu TRU tag duoc chi dinh
ansible-playbook site.yml --skip-tags install

# Liet ke Task se chay ma khong thuc thi (huu ich kiem tra tags/limit dung chua)
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-hosts
```

## 4. Xử lý lỗi khi chạy trên nhiều host

```bash
# Tiep tuc chay tren cac host con lai neu 1 host loi (mac dinh Ansible da lam vay)
# Nguoc lai, dung ngay lap tuc toan bo Playbook neu 1 host loi:
ansible-playbook site.yml --any-errors-fatal

# Chi chay lai tren cac host bi loi lan chay truoc (dung file .retry tu dong tao)
ansible-playbook site.yml --limit @site.retry
```

## 5. Debug và tăng độ chi tiết log

```bash
# Tang do chi tiet (-v den -vvvv, cang nhieu v cang chi tiet)
ansible-playbook site.yml -vv

# Buoc dung lai tai Task loi de nguoi van hanh quyet dinh (debugger)
ansible-playbook site.yml --step

# Bat dau chay tu mot Task cu the (bo qua cac Task truoc do)
ansible-playbook site.yml --start-at-task="Cai dat nginx"
```

## 6. Kiểm tra cú pháp trước khi chạy

```bash
# Chi kiem tra cu phap YAML/Ansible, khong ket noi managed node
ansible-playbook site.yml --syntax-check
```

Nên đưa `--syntax-check` vào bước CI trước khi merge Pull Request chứa thay đổi Playbook — bắt lỗi cú pháp sớm, không cần chờ tới lúc chạy thật trên môi trường test.

## 7. Bảng tổng hợp cờ dòng lệnh thường dùng

| Cờ | Chức năng |
|---|---|
| `--check` | Dry run, không thay đổi gì thật |
| `--diff` | Hiện chi tiết thay đổi (thường dùng cùng `--check`) |
| `--limit <pattern>` | Giới hạn host chạy |
| `--tags <tag>` | Chỉ chạy Task có tag |
| `--skip-tags <tag>` | Bỏ qua Task có tag |
| `--list-tasks` / `--list-hosts` | Xem trước, không thực thi |
| `--syntax-check` | Kiểm tra cú pháp |
| `-v` đến `-vvvv` | Tăng độ chi tiết log |
| `--start-at-task` | Bắt đầu từ Task cụ thể |
