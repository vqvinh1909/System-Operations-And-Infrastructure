---
title: "06 - Troubleshooting"
module: 3
tags: [ansible, sysops-infra, module-03, troubleshooting]
---

# 06 — Troubleshooting

## Lỗi 1: `ERROR! 'xxx' is not a valid attribute for a Task`

**Triệu chứng:** `ansible-playbook` báo lỗi ngay khi parse, chưa kết nối managed node nào.

**Nguyên nhân thường gặp:** thụt lề sai khiến một tham số của module bị đẩy lên cấp Task (ví dụ quên thụt lề `state: present` vào trong khối tham số module), hoặc gõ nhầm tên thuộc tính.

**Cách xử lý:**
```bash
# Luon chay syntax-check truoc khi debug sau hon
ansible-playbook site.yml --syntax-check
```
Đối chiếu cẩn thận cấp thụt lề giữa `name`, tên module, và các tham số bên trong module — lỗi YAML thụt lề (đã học ở Module 01) là nguyên nhân phổ biến nhất của nhóm lỗi này.

## Lỗi 2: Handler không chạy dù Task báo `changed`

**Triệu chứng:** Task copy file báo `changed: true` nhưng service không được restart như kỳ vọng.

**Nguyên nhân thường gặp:**
- Tên trong `notify:` không khớp **chính xác** (kể cả hoa/thường, khoảng trắng) với `name:` khai báo trong `handlers:`.
- Playbook dừng sớm ở một Task lỗi phía sau Task đã `notify`, khiến Ansible không bao giờ chạy tới bước flush handler ở cuối Play (Play bị dừng giữa chừng).
- Task nằm trong một `block` có `rescue`, và `rescue` không tự động trigger flush handler theo cách người mới thường kỳ vọng.

**Cách xử lý:**
```bash
# Chay voi -vv de xem Ansible co nhan dien notify va xep vao hang doi khong
ansible-playbook site.yml -vv | grep -i "notif\|handler"
```
Kiểm tra chính tả tên Handler bằng cách copy-paste thay vì gõ lại thủ công ở cả hai vị trí (`notify` và `handlers:`).

## Lỗi 3: `loop` báo lỗi `'item' is undefined` hoặc lặp sai giá trị

**Triệu chứng:** Task dùng `loop` báo biến `item` không tồn tại, hoặc mọi vòng lặp đều dùng cùng một giá trị.

**Nguyên nhân thường gặp:**
- Nhầm cú pháp cũ `with_items` với `loop` — hai cú pháp không hoàn toàn tương thích 1-1 trong mọi trường hợp phức tạp (ví dụ `with_dict` cần chuyển đổi khác khi sang `loop`).
- Dùng `loop` bên trong một `include_tasks`/`import_tasks` nhưng biến `item` bị che khuất (shadow) bởi vòng lặp ngoài cùng tên.

**Cách xử lý:** đổi tên biến lặp tường minh bằng `loop_control: loop_var: pkg` khi có vòng lặp lồng nhau, tránh xung đột tên `item` mặc định.

## Lỗi 4: Playbook dừng đột ngột ở một host, các host khác vẫn tiếp tục — hành vi có chủ đích hay lỗi?

**Triệu chứng:** một host báo `FAILED`, nhưng log cho thấy các host khác vẫn tiếp tục chạy phần còn lại của Playbook.

**Giải thích:** đây **không phải lỗi**, đây là hành vi mặc định của Ansible — mỗi host được xử lý độc lập; lỗi trên một host không tự động dừng các host khác (khác với suy nghĩ "một lỗi phải dừng tất cả" theo thói quen viết script tuần tự). Nếu nghiệp vụ yêu cầu dừng toàn bộ ngay khi có 1 host lỗi (ví dụ triển khai theo lô phải đồng bộ), dùng cờ `--any-errors-fatal` hoặc khai báo `any_errors_fatal: true` trong Play.

## Lỗi 5: `ignore_errors: true` khiến Playbook "chạy xong" nhưng hệ thống thực tế sai

**Triệu chứng:** Playbook báo hoàn tất không lỗi, nhưng kiểm tra thủ công phát hiện một bước quan trọng đã thất bại âm thầm.

**Nguyên nhân:** `ignore_errors: true` được đặt cho một Task quan trọng (không phải Task phụ) — che giấu lỗi thật thay vì chỉ bỏ qua lỗi không ảnh hưởng.

**Cách xử lý:** rà soát lại toàn bộ Playbook, chỉ giữ `ignore_errors: true` cho Task thực sự không quan trọng (ví dụ lệnh dọn log cũ, lỗi không ảnh hưởng kết quả cuối). Với Task quan trọng cần logic lỗi có kiểm soát, chuyển sang `block/rescue` (Module 03 mục 6) để xử lý lỗi tường minh thay vì im lặng bỏ qua.

## Lỗi 6: `import_tasks` với tên file chứa biến không hoạt động như mong đợi

**Triệu chứng:** `import_tasks: "{{ os_family }}.yml"` báo lỗi không tìm thấy file, dù file tồn tại.

**Nguyên nhân:** `import_tasks` resolve **lúc parse**, trước khi Ansible kết nối managed node và biết giá trị `os_family` — nên biến chưa có giá trị tại thời điểm cần tên file.

**Cách xử lý:** đổi sang `include_tasks`, resolve lúc chạy khi biến đã có giá trị — đúng như phân biệt đã học ở [[02-Theory|02 - Theory]] mục 10.
