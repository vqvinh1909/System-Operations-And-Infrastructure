---
title: "06 - Troubleshooting"
module: 4
tags: [ansible, sysops-infra, module-04, troubleshooting, jinja2]
---

# 06 — Troubleshooting

## Lỗi 1: `AnsibleUndefinedVariable: 'xxx' is undefined`

**Triệu chứng:** Template báo lỗi biến không tồn tại khi render trên một host cụ thể (nhưng chạy đúng trên host khác).

**Nguyên nhân:** biến được khai báo trong `host_vars/web-01.yml` nhưng quên khai báo cho `web-02` — Ansible không tự "suy luận" giá trị mặc định trừ khi bạn dùng filter `default`.

**Cách xử lý:**
```jinja2
{# Sai - crash neu host thieu bien nay #}
listen {{ http_port }};

{# Dung - co gia tri du phong an toan #}
listen {{ http_port | default(80) }};
```
Nguyên tắc thực tế: mọi biến "tùy chọn" (không phải mọi host đều cần khai báo riêng) trong template nên luôn có `default`, chỉ để biến "bắt buộc" (phải khác nhau theo host, không có giá trị mặc định hợp lý) crash khi thiếu — đây là tín hiệu tốt (fail fast) hơn là chạy với giá trị sai.

## Lỗi 2: `TemplateSyntaxError` — thiếu `{% endif %}` hoặc `{% endfor %}`

**Triệu chứng:** lỗi cú pháp Jinja2 khi render, thông báo dạng `unexpected end of template` hoặc `Encountered unknown tag`.

**Nguyên nhân:** quên đóng khối `{% if %}`/`{% for %}`, hoặc lồng nhiều khối điều kiện/vòng lặp mà đóng sai thứ tự.

**Cách xử lý:** kiểm tra lại cặp mở-đóng theo thứ tự từ trong ra ngoài — dùng editor có hỗ trợ Jinja2 syntax highlight để nhìn rõ cặp tag nào khớp cặp tag nào, đặc biệt quan trọng với template dài, nhiều tầng lồng nhau.

## Lỗi 3: `template` render ra nội dung đúng nhưng `owner`/`mode` sai

**Triệu chứng:** nội dung file cấu hình đúng như mong đợi, nhưng quyền file (`ls -l`) không đúng dự tính.

**Nguyên nhân:** quên khai báo `owner`/`group`/`mode` tường minh trong Task `template` — Ansible dùng umask mặc định của user chạy Task trên managed node, có thể khác nhau giữa các server (đặc biệt nếu server cấu hình umask không đồng nhất).

**Cách xử lý:** luôn khai báo tường minh cả 3 tham số `owner`, `group`, `mode` trong mọi Task `template`/`copy`/`file` — không phụ thuộc giá trị mặc định của hệ thống.

## Lỗi 4: Filter báo lỗi kiểu dữ liệu — `int object has no attribute 'append'`

**Triệu chứng:** filter như `join`, `map` báo lỗi liên quan kiểu dữ liệu không khớp.

**Nguyên nhân:** áp dụng filter dành cho list lên một biến thực chất là string hoặc int (thường do biến lấy từ `--extra-vars` dòng lệnh — mọi giá trị `--extra-vars` mặc định là string trừ khi dùng cú pháp JSON).

**Cách xử lý:**
```bash
# Sai - "packages" duoc hieu la 1 chuoi, khong phai list
ansible-playbook site.yml --extra-vars "packages=nginx,git,curl"

# Dung - truyen dung kieu list qua JSON
ansible-playbook site.yml --extra-vars '{"packages": ["nginx", "git", "curl"]}'
```

## Lỗi 5: `template` báo `changed` mỗi lần chạy dù nội dung không đổi

**Triệu chứng:** vi phạm idempotency — Task `template` luôn báo `changed: true` dù không có gì thay đổi thực sự.

**Nguyên nhân thường gặp:** template chứa giá trị **luôn thay đổi mỗi lần render** — ví dụ vô tình dùng `lookup('pipe', 'date')` (không có format cố định, ra giây hiện tại) trực tiếp trong nội dung file thay vì chỉ trong tên file backup, khiến nội dung file luôn khác timestamp cũ.

**Cách xử lý:** rà soát template, đảm bảo mọi giá trị động (timestamp, random) chỉ nằm ở nơi **cần** thay đổi (tên file, comment ghi chú lần deploy gần nhất — nếu chấp nhận được) chứ không lẫn vào phần nội dung cấu hình cốt lõi cần ổn định giữa các lần chạy.

## Lỗi 6: `synchronize` báo lỗi `rsync: command not found`

**Triệu chứng:** Task `synchronize` thất bại với lỗi liên quan không tìm thấy lệnh `rsync`.

**Nguyên nhân:** `synchronize` là wrapper gọi `rsync` thật bên dưới — cần `rsync` được cài **cả trên Control Node và Managed Node**, khác với `copy`/`template` không cần thêm phần mềm ngoài.

**Cách xử lý:** thêm Task cài `rsync` trước (hoặc xác nhận đã có trong base image) trước khi dùng `synchronize`, đặc biệt với managed node dùng image tối giản (minimal container/cloud image).
