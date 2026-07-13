---
title: "01 - Introduction"
module: 3
tags: [ansible, sysops-infra, module-03, introduction]
---

# 01 — Giới thiệu

## Bài toán: khi ad-hoc command không còn đủ

Ở Module 02, bạn cài `nginx` bằng một dòng ad-hoc: `ansible webservers -b -m apt -a "name=nginx state=present"`. Nhưng triển khai một web server thật cần nhiều hơn một bước: cài package, tạo thư mục cấu hình, copy file config, mở firewall, start service, kiểm tra service đã chạy đúng — và nếu bước cài config thất bại giữa chừng, bạn cần dừng lại thay vì tiếp tục các bước sau trên một hệ thống dở dang.

Bạn **có thể** gõ 6 dòng ad-hoc liên tiếp — nhưng: không có gì đảm bảo bạn (hoặc đồng nghiệp 6 tháng sau) nhớ đúng thứ tự, không lưu lại được để review trong Pull Request, không chạy được trong CI/CD, và không có cơ chế "nếu bước 3 lỗi thì dừng lại" built-in. **Playbook** giải quyết chính xác những vấn đề này.

## Playbook là gì

Playbook là một file YAML mô tả **một chuỗi Task có thứ tự**, áp dụng lên một hoặc nhiều nhóm host, có thể chứa biến, điều kiện, vòng lặp, xử lý lỗi — và quan trọng nhất: **có thể lưu vào Git, review qua Pull Request, chạy lại nhiều lần với kết quả nhất quán (idempotent), và tái sử dụng**.

```yaml
---
- name: Trien khai web server co ban
  hosts: webservers
  become: true
  tasks:
    - name: Cai dat nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Dam bao nginx dang chay va tu khoi dong cung he thong
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: true
```

Chỉ 2 task đơn giản này đã thể hiện toàn bộ triết lý Playbook: mô tả **trạng thái mong muốn** ("nginx phải được cài, phải đang chạy, phải tự khởi động"), không phải "các bước phải làm" — nhất quán với tư duy Declarative đã học ở Module 00.

## Vì sao Playbook là kỹ năng trung tâm, không phải kỹ năng phụ

Trong thực tế vận hành doanh nghiệp, gần như 100% công việc Ansible hàng ngày của một kỹ sư là **đọc, viết, sửa Playbook** — ad-hoc command (Module 02) chỉ chiếm phần nhỏ cho debug/kiểm tra nhanh. Mọi khái niệm nâng cao ở các module sau (Template, Vault, Role) đều là cách **tổ chức và tái sử dụng** nội dung bên trong hoặc xung quanh Playbook, không phải khái niệm tách biệt.

## Mục tiêu học tập

Sau module này, bạn viết được một Playbook nhiều Task hoàn chỉnh, biết dùng Loop để tránh lặp code, Conditional để xử lý sự khác biệt giữa các host, Handler để tối ưu việc restart service, Block để nhóm task và xử lý lỗi có cấu trúc, và Tags để chạy chọn lọc từng phần — đủ năng lực viết Playbook triển khai một dịch vụ thật, chuẩn bị cho Module 04 (Templates) nơi bạn sẽ render file cấu hình động thay vì copy file tĩnh.
