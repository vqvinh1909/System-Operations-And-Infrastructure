---
title: "07 - Interview"
module: 1
tags: [ansible, sysops-infra, module-01, interview]
---

# 07 — Câu hỏi phỏng vấn

## Junior

**1. Vì sao Ansible bắt buộc dùng SSH key-based authentication thay vì password?**

> Đáp án: Ansible thường chạy hàng loạt task trên nhiều host liên tiếp trong một lần thực thi — không thể có người ngồi gõ password cho từng kết nối. SSH key cho phép xác thực tự động, không tương tác (non-interactive), là điều kiện bắt buộc để automation hoạt động ở quy mô, đồng thời an toàn hơn password (không sợ brute-force, không lộ khi gõ nhầm).

**2. `~/.ssh/config` dùng để làm gì? Cho một ví dụ khối cấu hình đơn giản.**

> Đáp án: Cho phép đặt alias và cấu hình riêng (user, key, port) cho từng host hoặc nhóm host theo pattern, tránh phải gõ đầy đủ tham số SSH mỗi lần kết nối. Ví dụ:
> ```
> Host web-01
>     HostName 10.0.1.11
>     User deploy
>     IdentityFile ~/.ssh/id_automation
> ```

**3. YAML dùng gì để thể hiện quan hệ cha-con (nesting)?**

> Đáp án: Thụt lề (indentation) bằng space — không dùng tab. Ansible cộng đồng khuyến nghị 2 space mỗi cấp.

**4. Sự khác nhau giữa `{{ ... }}` và `{% ... %}` trong Jinja2?**

> Đáp án: `{{ ... }}` dùng để xuất ra giá trị của một biến hoặc biểu thức. `{% ... %}` dùng cho câu lệnh điều khiển như vòng lặp (`for`) hoặc điều kiện (`if`), không trực tiếp xuất giá trị ra output.

## Mid-level

**5. Tại sao nên tạo một SSH key riêng cho automation, tách biệt với key cá nhân của kỹ sư?**

> Đáp án: Key cá nhân gắn với danh tính một người — khi người đó rời team hoặc cần thu hồi quyền, việc revoke key sẽ ảnh hưởng luôn tới mọi automation đang dùng chung key đó. Key automation là tài sản của hệ thống/team, được quản lý vòng đời độc lập (rotate, revoke, audit) không phụ thuộc nhân sự cụ thể — đây cũng là nguyên tắc tách biệt trách nhiệm (separation of duty) trong vận hành doanh nghiệp.

**6. Thiết kế sudo cho service account automation khác gì so với sudo cho user thông thường? Rủi ro của cấu hình `NOPASSWD: ALL` không giới hạn là gì?**

> Đáp án: Service account cần NOPASSWD vì automation chạy không tương tác, nhưng `NOPASSWD: ALL` không giới hạn nghĩa là nếu key automation bị lộ, kẻ tấn công có toàn quyền root trên mọi managed node ngay lập tức, không cần biết thêm bất kỳ credential nào khác. Cấu hình trưởng thành hơn giới hạn NOPASSWD chỉ cho các lệnh cụ thể cần thiết, kết hợp logging chi tiết (`Defaults log_output`) để audit hành động của automation.

**7. Vì sao `msg: {{ ten_bien }}` (không có ngoặc kép) trong một file YAML thường gây lỗi cú pháp, còn `msg: "{{ ten_bien }}"` thì không?**

> Đáp án: YAML có cú pháp flow-mapping riêng dùng cặp `{ }` để viết dictionary trên một dòng. Khi gặp `{{ ` không có ngoặc kép bao ngoài, YAML parser cố hiểu đây là mở flow-mapping thay vì cú pháp Jinja2, dẫn tới lỗi vì nội dung bên trong không hợp lệ theo cú pháp flow-mapping. Đặt trong ngoặc kép biến toàn bộ thành một chuỗi (scalar) hợp lệ theo YAML, Jinja2 render sau khi YAML đã parse xong.

**8. Bạn thiết kế inventory cho 20 server: 10 web, 5 db, 5 cache, trải trên 2 datacenter, 2 môi trường (prod/staging). Bạn sẽ phân nhóm theo những trục nào, vì sao?**

> Đáp án: Ba trục độc lập — theo vai trò (`webservers`, `dbservers`, `cacheservers`), theo môi trường (`production`, `staging`), theo vị trí (`dc-hanoi`, `dc-hcm`). Một host thuộc đồng thời nhiều nhóm (ví dụ `web-01` vừa trong `webservers`, vừa trong `production`, vừa trong `dc-hanoi`). Thiết kế đa trục này cho phép áp dụng thay đổi linh hoạt theo bất kỳ tổ hợp điều kiện nào, thay vì phải duy trì danh sách host phẳng và lặp lại logic phân loại thủ công.

## Thực chiến

**9. Bạn SSH thủ công vào một server bằng key cá nhân thành công, nhưng Ansible báo `UNREACHABLE` khi target đúng server đó qua key automation. Bạn debug theo thứ tự nào?**

> Đáp án: (1) Xác nhận đang thật sự dùng đúng key automation, không nhầm sang key mặc định — kiểm tra bằng `ssh -i ~/.ssh/id_automation -vvv user@host` chạy thủ công trước khi đổ lỗi cho Ansible. (2) Kiểm tra `authorized_keys` trên managed node có đúng public key tương ứng chưa, và quyền file/thư mục `.ssh` (700/600) có đúng không. (3) Kiểm tra user Ansible đang dùng để kết nối (`ansible_user` trong inventory, học ở Module 02) có khớp với user đã cài key hay không. (4) Nếu vẫn lỗi, kiểm tra `known_hosts` — nếu chạy trong CI/CD không tương tác, việc chờ xác nhận fingerwork mới cũng biểu hiện tương tự treo/lỗi kết nối.

**10. Một đồng nghiệp báo YAML inventory "tự nhiên" chạy sai — một host bị gán nhầm sang nhóm khác dù không ai chủ động sửa nhóm đó. Nguyên nhân khả dĩ nhất liên quan tới YAML là gì, và làm sao xác minh?**

> Đáp án: Khả năng cao nhất là lỗi thụt lề lệch 1 space khiến một dòng bị hiểu nhầm là con của node khác (ví dụ một host tưởng thuộc `webservers` nhưng do thụt lề sai lại nằm dưới `dbservers` trong cấu trúc dữ liệu thực tế), YAML không báo lỗi cú pháp vì cấu trúc vẫn "hợp lệ" — chỉ sai ý định. Xác minh bằng cách chạy `python3 -c "import yaml, pprint; pprint.pprint(yaml.safe_load(open('file.yml')))"` để in ra cấu trúc dictionary/list Python thực sự được parse, so sánh trực tiếp với ý định ban đầu thay vì tin vào cách file "trông" đúng trên editor.

**11. Vì sao module này không dạy lại toàn bộ SSH/sudo từ đầu mà chỉ "ôn nhanh"? Điều đó nói lên điều gì về cách tổ chức một giáo trình/dự án lớn trong thực tế?**

> Đáp án: Vì kiến thức nền (SSH, sudo, filesystem) đã có sẵn nguồn tham chiếu chất lượng (khóa Linux Sysadmin trước đó) — lặp lại toàn bộ vừa lãng phí thời gian, vừa tạo nguy cơ hai bản tài liệu lệch nhau khi một bên được cập nhật. Nguyên tắc "single source of truth, tham chiếu ngược thay vì sao chép" áp dụng y hệt trong tài liệu kỹ thuật doanh nghiệp thực tế (runbook, wiki nội bộ) — viết một nơi, liên kết nhiều nơi, tránh nội dung trùng lặp không đồng bộ.
