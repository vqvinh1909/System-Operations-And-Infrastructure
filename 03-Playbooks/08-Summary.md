---
title: "08 - Summary"
module: 3
tags: [ansible, sysops-infra, module-03, summary]
---

# 08 — Tóm tắt Module 03

## Những gì đã học

- **Cấu trúc Playbook:** Playbook chứa nhiều Play, mỗi Play có `hosts`, `become`, `vars`, `tasks`.
- **Loops:** `loop` + biến `item` để tránh lặp code.
- **Conditionals:** `when` chạy Task có điều kiện, thường kết hợp Facts hoặc `register`.
- **Handlers:** chỉ chạy khi có `notify` từ Task `changed: true`, mặc định chạy một lần ở cuối Play.
- **Blocks:** `block/rescue/always` — mô hình try/except/finally, công cụ xử lý lỗi mạnh nhất.
- **Error Handling đơn giản:** `ignore_errors` (bỏ qua lỗi), `failed_when` (định nghĩa lại thế nào là lỗi).
- **Tags:** chạy chọn lọc bằng `--tags`/`--skip-tags`.
- **Include vs Import:** Include resolve lúc chạy (hỗ trợ biến động), Import resolve lúc parse (nhanh hơn, không hỗ trợ tên file động).

## Bảng đối chiếu công cụ xử lý lỗi

| Công cụ | Mức độ kiểm soát | Dùng khi |
|---|---|---|
| Mặc định (không xử lý gì) | Thấp — Playbook dừng ở host lỗi | Lỗi phải chặn ngay, không có phương án khắc phục |
| `ignore_errors: true` | Trung bình — bỏ qua, tiếp tục | Task phụ, lỗi không ảnh hưởng kết quả cuối |
| `failed_when` | Trung bình — định nghĩa lại điều kiện lỗi | Module trả `rc=0` nhưng nghiệp vụ coi là thất bại |
| `block/rescue/always` | Cao — có logic khắc phục + dọn dẹp | Cần rollback hoặc đảm bảo hành động luôn chạy |

## Điều kiện để chuyển sang Module 04

1. Viết được Playbook nhiều Task, chạy idempotent?
2. Dùng thành thạo `loop` và `when` cho tình huống thực tế (nhiều package, nhiều hệ điều hành)?
3. Giải thích được (không cần tra lại tài liệu) vì sao Handler chỉ chạy một lần ở cuối Play?
4. Viết được `block/rescue/always` xử lý lỗi có cấu trúc?
5. Phân biệt rõ khi nào dùng `import_tasks`, khi nào dùng `include_tasks`?

## Cầu nối sang Module 04

Playbook ở module này mới chỉ **copy file tĩnh** (`copy` module) cho cấu hình. Trong thực tế, file cấu hình gần như luôn cần khác nhau giữa các host (IP, port, tên miền riêng từng môi trường) — Module 04 dạy **Jinja2 Templates**, cho phép một file cấu hình duy nhất tự động render ra nội dung khác nhau cho từng host dựa trên Variables/Facts, thay thế hoàn toàn cách copy file tĩnh cứng nhắc.

## Điều hướng

- Quay lại: [[README|Module 03 — README]]
- Tiếp theo: [[../04-Templates/README|Module 04 — Templates]]
