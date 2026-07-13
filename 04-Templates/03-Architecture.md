---
title: "03 - Architecture"
module: 4
tags: [ansible, sysops-infra, module-04, architecture, diagram, jinja2]
---

# 03 — Kiến trúc & Sơ đồ

## Sơ đồ 1: Luồng render Template — từ file `.j2` tới file cấu hình thật

```
+----------------------------------------+
|     nginx.conf.j2 (co {{ bien }})      |
+----------------------------------------+
                     |
                     v
+----------------------------------------+
|  Ansible doc bien tu Inventory/Facts   |
|        cho tung host rieng biet        |
+----------------------------------------+
                     |
                     v
+----------------------------------------+
| Jinja2 Engine render tren Control Node |
+----------------------------------------+
                     |
                     v
+----------------------------------------+
|File .conf hoan chinh (khong con {{ }}) |
+----------------------------------------+
                     |
                     v
+----------------------------------------+
|      Ghi len Managed Node qua SSH      |
+----------------------------------------+
```

Đọc sơ đồ: điểm mấu chốt là **Jinja2 render xảy ra trên Control Node**, trước khi bất kỳ dữ liệu nào được gửi qua SSH — managed node chỉ nhận file cấu hình đã hoàn chỉnh, không có khái niệm `{{ }}` nào còn sót lại. Đây là lý do managed node không cần cài Jinja2 hay bất kỳ engine template nào — nhất quán với kiến trúc agentless đã học ở Module 02.

## Sơ đồ 2: Chiều dữ liệu của 4 module xử lý file

```
+------------------+                         +------------------+
|   CONTROL NODE   |                         |   MANAGED NODE   |
+------------------+                         +------------------+

  copy / template  : CONTROL NODE  ---------> MANAGED NODE
                     (mot chieu, day cau hinh/ma nguon xuong)

  fetch            : CONTROL NODE  <--------- MANAGED NODE
                     (mot chieu, keo log/artifact ve)

  synchronize      : CONTROL NODE  <--------> MANAGED NODE
                     (hai chieu tuy tham so, dong bo qua rsync)
```

Đọc sơ đồ: `copy`/`template` luôn đẩy dữ liệu **từ Control Node xuống Managed Node** — dùng cho triển khai cấu hình/mã nguồn. `fetch` đi **ngược lại**, kéo dữ liệu từ Managed Node về Control Node — dùng cho thu thập log/artifact. `synchronize` (dựa trên rsync) hỗ trợ cả hai chiều tùy tham số, tối ưu cho lượng dữ liệu lớn nhờ cơ chế đồng bộ delta thay vì truyền lại toàn bộ file. `file` không nằm trong sơ đồ này vì không di chuyển dữ liệu — chỉ quản lý thuộc tính file đã tồn tại trên Managed Node.
