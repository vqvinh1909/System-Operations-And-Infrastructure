---
title: "07 - Interview"
module: 6
tags: [ansible, sysops-infra, module-06, interview]
---

# 07 — Câu hỏi phỏng vấn

## Junior

**1. Role trong Ansible là gì?**

> Đáp án: Một cấu trúc thư mục chuẩn hóa đóng gói Task, Handler, Template, Variables liên quan tới một chức năng cụ thể, giúp tái sử dụng giữa nhiều Playbook/dự án.

**2. Lệnh nào tạo khung thư mục Role chuẩn?**

> Đáp án: `ansible-galaxy role init <ten-role>`.

**3. Kể tên ít nhất 4 thư mục con chuẩn của một Role.**

> Đáp án: `tasks`, `handlers`, `templates`, `files`, `vars`, `defaults`, `meta` (bất kỳ 4 trong số này).

**4. `defaults/main.yml` và `vars/main.yml` khác nhau ở điểm nào cơ bản nhất?**

> Đáp án: `defaults` có độ ưu tiên thấp nhất, dành cho giá trị người dùng Role có thể ghi đè. `vars` có độ ưu tiên cao, dành cho giá trị nội bộ không nên bị ghi đè tùy tiện.

**5. Ansible Galaxy dùng để làm gì?**

> Đáp án: Kho lưu trữ công khai để tìm, chia sẻ, tải Role và Collection do cộng đồng/tổ chức đóng góp, tránh phải viết lại logic phổ biến từ đầu.

## Mid-level

**6. Giải thích ngắn gọn quy ước "convention over configuration" áp dụng như thế nào trong cấu trúc thư mục Role?**

> Đáp án: Ansible tự động nhận diện chức năng của từng thư mục con dựa trên **tên thư mục cố định** (`tasks/`, `handlers/`, `defaults/`...) mà không cần khai báo cấu hình thủ công chỉ định "file này là Task, file kia là Handler". Nhờ quy ước chuẩn hóa này, bất kỳ kỹ sư Ansible nào cũng đọc hiểu ngay cấu trúc một Role lạ mà không cần tài liệu giải thích thêm.

**7. Role Dependencies (`meta/main.yml`) khác gì so với việc gọi nhiều Role tuần tự trong `roles:` của Playbook?**

> Đáp án: Dependencies được khai báo **bên trong** Role, tự động chạy trước Role đó mỗi khi Role được gọi ở bất kỳ đâu — đảm bảo logic phụ thuộc luôn nhất quán, không phụ thuộc vào việc người viết Playbook có nhớ khai báo đúng thứ tự hay không. Gọi nhiều Role tuần tự trong `roles:` của Playbook đòi hỏi người viết Playbook phải tự biết và tự sắp đúng thứ tự — dễ sai sót khi Playbook được viết bởi người không hiểu rõ nội bộ từng Role.

**8. Vì sao nên đặt tên biến có tiền tố tên Role (ví dụ `nginx_port` thay vì `port`)?**

> Đáp án: Vì nhiều Role có thể cùng chạy trong một Playbook — nếu dùng tên biến chung chung không tiền tố, nguy cơ một Role vô tình ghi đè biến của Role khác (do trùng tên trong cùng không gian biến của Ansible) rất cao. Tiền tố tên Role là quy ước tối thiểu để tránh xung đột.

**9. Role và Collection khác nhau như thế nào về phạm vi đóng gói?**

> Đáp án: Role đóng gói một chức năng cụ thể (Task/Template/Variables cho một mục đích). Collection có phạm vi rộng hơn, đóng gói nhiều Module, Plugin, Filter, Lookup, và có thể chứa cả nhiều Role, dưới một namespace thống nhất `<tổ_chức>.<tên>`.

## Thực chiến

**10. Team bạn có 3 Role: `common`, `webserver`, `database`, cả `webserver` và `database` đều phụ thuộc `common` (cấu hình hardening chung). Một kỹ sư mới thêm một Role thứ 4 `monitoring`, và vô tình khai báo `monitoring` phụ thuộc `webserver`, còn `webserver` lại được sửa để phụ thuộc `monitoring` (do cần agent monitoring cài trước khi mở port). Vấn đề gì sẽ xảy ra, và bạn review Pull Request này như thế nào?**

> Đáp án: Đây là circular dependency (`webserver` → `monitoring` → `webserver`) — Ansible sẽ báo lỗi hoặc treo khi cố resolve thứ tự chạy. Khi review, cần chỉ ra vấn đề kiến trúc: không nên để 2 Role phụ thuộc lẫn nhau. Giải pháp đúng: tách phần "cài agent monitoring cơ bản" thành một Role con độc lập (ví dụ `monitoring_agent_base`) mà cả `webserver` và `monitoring` cùng phụ thuộc, thay vì để chúng phụ thuộc trực tiếp lẫn nhau — loại bỏ hoàn toàn chu trình khép kín.

**11. Bạn đang thiết kế Role `postgresql_ha` cho cụm PostgreSQL có 3 node (1 primary, 2 replica), cần biết node nào đóng vai trò gì. Bạn sẽ thiết kế biến cấu hình này ở `defaults` hay `vars`? Giải thích cách bạn sẽ cho phép người dùng Role chỉ định vai trò từng node.**

> Đáp án: Vai trò từng node (primary/replica) là giá trị **bắt buộc phải khác nhau theo từng host**, không có "giá trị mặc định hợp lý chung" — nên đây là ứng viên tốt để khai báo qua `host_vars` (ví dụ `postgresql_role: primary` cho node 1, `postgresql_role: replica` cho 2 node còn lại), không nên đặt cứng trong `defaults` hay `vars` của Role. Trong Role, `defaults/main.yml` chỉ nên chứa giá trị mặc định chung không phụ thuộc vai trò (ví dụ `postgresql_port: 5432`), còn Task bên trong Role dùng `when: postgresql_role == "primary"` để rẽ nhánh logic theo `host_vars` đã khai báo — tách bạch rõ "cấu hình chung của Role" và "cấu hình riêng theo host" đúng tinh thần Variable Precedence đã học ở Module 05.

**12. Sếp yêu cầu bạn đánh giá rủi ro khi team dùng Role `geerlingguy.mysql` từ Ansible Galaxy cho database production, thay vì tự viết Role nội bộ. Bạn trình bày những điểm cần cân nhắc nào?**

> Đáp án mẫu: Lợi ích: tiết kiệm thời gian phát triển, Role phổ biến thường đã qua kiểm thử rộng rãi bởi cộng đồng, ít lỗi cơ bản hơn Role tự viết vội. Rủi ro cần đánh giá: (1) Mức độ bảo trì — Role có còn active maintain không, có phản hồi issue nhanh không. (2) Ghim version cụ thể trong `requirements.yml`, không tự động lấy bản mới nhất — tránh thay đổi hành vi bất ngờ. (3) Review kỹ mã nguồn Role trước khi dùng cho production, đặc biệt các Task có `become: true` — không tin tưởng mù quáng vào số lượt tải/star. (4) Đánh giá mức độ Role đó có phù hợp với chuẩn bảo mật nội bộ công ty hay không — nếu không, cân nhắc dùng Role đó làm nền, wrap thêm một Role nội bộ mỏng bên ngoài để áp thêm hardening riêng thay vì sửa trực tiếp code của Role bên thứ ba (khó maintain khi upstream cập nhật).
