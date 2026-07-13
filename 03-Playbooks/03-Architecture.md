---
title: "03 - Architecture"
module: 3
tags: [ansible, sysops-infra, module-03, architecture, diagram, handlers, blocks]
---

# 03 — Kiến trúc & Sơ đồ

## Sơ đồ 1: Cấu trúc phân cấp Playbook — Play — Task

```
+--------------------------------------+
|         PLAYBOOK (file .yml)         |
+--------------------------------------+
                    |
        +-----------+-----------+
        |                       |
        v                       v
+--------------------------------------+
|      Play 1: hosts: webservers       |
+--------------------------------------+
+--------------------------------------+
|       Play 2: hosts: dbservers       |
+--------------------------------------+
```

Đọc sơ đồ: một Playbook chứa nhiều Play, mỗi Play chạy tuần tự theo thứ tự khai báo, nhắm vào một nhóm host khác nhau (lấy từ Inventory, đã học ở Module 02). Bên trong mỗi Play là danh sách Task chạy tuần tự — không vẽ riêng vì đã minh họa qua ví dụ code trong [[02-Theory|02 - Theory]].

## Sơ đồ 2: Vòng đời Handler — vì sao chỉ chạy 1 lần ở cuối Play

```
+------------------------------------------+
|         Task 1: Copy nginx.conf          |
|          notify: Restart nginx           |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|         Task 2: Copy index.html          |
|              (khong notify)              |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|          Task 3: Copy ssl.conf           |
|          notify: Restart nginx           |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|          HANDLER QUEUE (noi bo)          |
|     Restart nginx  <- chi xep 1 lan      |
+------------------------------------------+
                     |
                     v
+------------------------------------------+
|        Cuoi Play: flush handlers         |
|    Restart nginx CHAY DUY NHAT 1 LAN     |
+------------------------------------------+
```

Đọc sơ đồ: Task 1 và Task 3 đều `notify: Restart nginx`, nhưng Handler `Restart nginx` chỉ được **xếp vào hàng đợi nội bộ một lần** (Ansible tự loại trùng), và chỉ thực thi **một lần duy nhất ở cuối Play** — dù bị notify bao nhiêu lần trong Play đó. Nếu không có Task nào notify (như Task 2), Handler không chạy. Đây là lý do restart service chỉ xảy ra đúng một lần dù có nhiều thay đổi cấu hình trong cùng một lần chạy Playbook.

## Sơ đồ 3: Block — Rescue — Always (mô hình try/except/finally)

```
+--------------------------------------+
| block:                               |
| - Dung service                       |
| - Cap nhat ma nguon                  |
+--------------------------------------+
                    |
        +-----------+-----------+
        | Thanh cong            | That bai
        v                       v
   (bo qua rescue)   +--------------------------------------+
        |             | rescue:                              |
        |             | - Rollback ve ban cu                 |
        |             +--------------------------------------+
        |                       |
        +-----------+-----------+
                    v
+--------------------------------------+
| always:                              |
| - Khoi dong lai service              |
+--------------------------------------+
```

Đọc sơ đồ: nếu mọi Task trong `block` thành công, Ansible **bỏ qua hoàn toàn** `rescue` và đi thẳng tới `always`. Nếu bất kỳ Task nào trong `block` thất bại, Ansible dừng ngay `block`, chạy `rescue` để khắc phục/rollback, rồi vẫn tiếp tục tới `always`. `always` **luôn luôn chạy** bất kể nhánh nào ở trên đã xảy ra — đảm bảo bước dọn dẹp/khôi phục trạng thái (ví dụ khởi động lại service) không bao giờ bị bỏ sót.

## Sơ đồ 4: Include (động) vs Import (tĩnh) — thời điểm resolve

```
+----------------------------------------------+
| IMPORT: resolve luc PARSE (truoc khi chay)   |
|   import_tasks: file.yml                     |
|   -> Ansible doc va "nhung" noi dung file    |
|      vao Playbook ngay khi doc, truoc Task 1 |
+----------------------------------------------+

+----------------------------------------------+
| INCLUDE: resolve luc CHAY (runtime)          |
|   include_tasks: "{{ os_family }}.yml"       |
|   -> Ansible cho toi dung dong nay moi doc   |
|      file, gia tri bien da co san luc do     |
+----------------------------------------------+
```

Đọc sơ đồ: khác biệt thời điểm resolve giải thích trực tiếp vì sao `import_tasks` **không hỗ trợ tên file chứa biến động** tốt (lúc parse, biến có thể chưa có giá trị), trong khi `include_tasks` hỗ trợ đầy đủ (lúc chạy, mọi biến/facts đã sẵn sàng) — đúng như bảng so sánh ở [[02-Theory|02 - Theory]] mục 10.
