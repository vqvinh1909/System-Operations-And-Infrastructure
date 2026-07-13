---
title: "07 - Interview"
module: 4
tags: [ansible, sysops-infra, module-04, interview]
---

# 07 — Câu hỏi phỏng vấn

## Junior

**1. Module `template` khác `copy` ở điểm nào?**

> Đáp án: `template` render cú pháp Jinja2 (`{{ }}`, `{% %}`) trong file trước khi ghi lên managed node, cho phép nội dung khác nhau theo biến/host. `copy` chép nguyên văn, không xử lý Jinja2.

**2. Filter trong Jinja2 dùng ký hiệu gì? Cho 2 ví dụ filter thường dùng.**

> Đáp án: Dùng ký hiệu `|`. Ví dụ: `default` (giá trị dự phòng), `upper` (chuyển hoa).

**3. Lookup Plugin thực thi trên máy nào — Control Node hay Managed Node?**

> Đáp án: Control Node.

**4. Module nào dùng để lấy file từ Managed Node về Control Node?**

> Đáp án: `ansible.builtin.fetch`.

**5. Vì sao nên khai báo tường minh `owner`/`group`/`mode` trong Task `template`/`copy`?**

> Đáp án: Tránh phụ thuộc vào umask mặc định của hệ thống (có thể khác nhau giữa các managed node), đảm bảo quyền file nhất quán và có thể dự đoán trước.

## Mid-level

**6. Giải thích vì sao `synchronize` thường nhanh hơn `copy` khi triển khai một thư mục nhiều file?**

> Đáp án: `synchronize` dùng `rsync` bên dưới, chỉ truyền phần dữ liệu thay đổi (delta) so với lần đồng bộ trước, trong khi `copy` không có cơ chế delta — mỗi lần chạy đều so sánh/ghi từng file riêng lẻ qua giao thức SFTP/SCP của SSH, chậm hơn đáng kể với số lượng file lớn.

**7. Bạn dùng `--extra-vars "packages=nginx,git,curl"` và filter `join` báo lỗi kiểu dữ liệu. Giải thích nguyên nhân và cách sửa.**

> Đáp án: Mọi giá trị truyền qua `--extra-vars` dạng `key=value` mặc định được hiểu là string, không tự động thành list — nên `packages` thực chất là chuỗi `"nginx,git,curl"`, không phải list 3 phần tử, khiến filter dành cho list (`join`, `map`...) lỗi kiểu dữ liệu. Sửa bằng cách truyền `--extra-vars` dạng JSON: `--extra-vars '{"packages": ["nginx","git","curl"]}'`.

**8. Vì sao nên dùng filter `default()` cho phần lớn biến "tùy chọn" trong template, nhưng không nên dùng cho mọi biến?**

> Đáp án: `default()` giúp template không crash khi một host thiếu khai báo biến không bắt buộc, tăng tính linh hoạt và giảm lỗi vận hành. Tuy nhiên với biến thực sự bắt buộc phải khác nhau theo host (ví dụ `domain_name`), dùng `default()` che giấu lỗi cấu hình thật — nên để nó crash (fail fast) để phát hiện sớm host nào thiếu khai báo quan trọng, thay vì âm thầm chạy với giá trị mặc định sai.

**9. `validate` trong module `template` dùng để làm gì, và vì sao quan trọng khi triển khai cấu hình dịch vụ production?**

> Đáp án: `validate` chạy một lệnh kiểm tra cú pháp file tạm **trước khi** ghi đè file thật (ví dụ `nginx -t -c %s`). Quan trọng vì nếu template render ra một file cấu hình sai cú pháp (do lỗi logic Jinja2 hoặc biến sai), Ansible sẽ dừng lại và không ghi đè — tránh tình huống service không khởi động lại được sau khi Handler restart chạy trên một file cấu hình lỗi.

## Thực chiến

**10. Bạn triển khai template `nginx.conf.j2` lên 30 server, sau khi chạy phát hiện 28 server render đúng nhưng 2 server có nội dung file sai (thiếu khối `upstream`). Bạn debug thế nào, khả năng nguyên nhân là gì?**

> Đáp án: Trước tiên kiểm tra biến `upstream_servers` (list dùng trong `{% for %}`) có được khai báo đầy đủ cho 2 host bị lỗi hay không — dùng `ansible-inventory --host <ten-host>` để xem toàn bộ biến đã resolve cho host đó. Khả năng cao: 2 host này thiếu khai báo `upstream_servers` trong `host_vars`, khiến `{% for %}` lặp trên list rỗng (không lỗi cú pháp, chỉ render ra khối rỗng) — đây là ví dụ điển hình lỗi logic template không gây crash, chỉ phát hiện được qua review nội dung thực tế, nhấn mạnh tầm quan trọng của `--check --diff` trước khi áp dụng production.

**11. Một đồng nghiệp đề xuất dùng `lookup('pipe', 'openssl rand -hex 16')` trực tiếp trong template để sinh giá trị bí mật ngẫu nhiên (ví dụ session secret) cho file cấu hình ứng dụng. Bạn thấy vấn đề gì với cách làm này?**

> Đáp án: Vấn đề lớn nhất: `lookup('pipe', ...)` chạy lại **mỗi lần** Playbook được thực thi, sinh ra giá trị ngẫu nhiên **mới** mỗi lần — vi phạm idempotency, khiến file cấu hình luôn báo `changed` và giá trị bí mật đổi liên tục (có thể làm session hiện tại của người dùng bị vô hiệu mỗi lần chạy Playbook, hoặc gây lỗi nếu nhiều service cần cùng một giá trị bí mật đồng bộ). Cách đúng: sinh giá trị bí mật **một lần**, lưu cố định (mã hóa bằng Ansible Vault — Module 05), rồi tham chiếu lại giá trị đã lưu trong mọi lần chạy sau, không sinh lại từ đầu mỗi lần.

**12. Sếp yêu cầu bạn giải thích ngắn gọn: "Tại sao chúng ta cần cả `copy` lẫn `template`, sao không dùng `template` cho mọi trường hợp luôn cho đơn giản?"**

> Đáp án mẫu: "Về mặt kỹ thuật `template` xử lý được cả trường hợp `copy` (nếu file không chứa cú pháp Jinja2, `template` vẫn chạy đúng), nhưng có hai lý do nên tách bạch: (1) Hiệu năng — `template` luôn chạy qua bước phân tích Jinja2 dù không cần thiết, chậm hơn `copy` khi xử lý số lượng lớn file tĩnh không đổi (ví dụ static asset, ảnh, font). (2) Rõ ràng ý định — khi đọc Playbook, thấy `copy` nghĩa là 'file này cố định, không phụ thuộc biến gì', thấy `template` nghĩa là 'file này có logic động, cần xem file `.j2` gốc để hiểu đầy đủ' — phân biệt module giúp code dễ đọc và bảo trì hơn, đặc biệt khi Playbook lớn có hàng chục Task."
