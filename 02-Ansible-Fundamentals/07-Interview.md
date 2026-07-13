---
title: "07 - Interview"
module: 2
tags: [ansible, sysops-infra, module-02, interview]
---

# 07 — Câu hỏi phỏng vấn

## Junior

**1. Ansible là gì? Nó thuộc loại công cụ nào trong hệ sinh thái DevOps?**

> Đáp án: Ansible là công cụ automation mã nguồn mở dùng cho Configuration Management, Application Deployment, và Orchestration, hoạt động theo kiến trúc agentless (không cần cài daemon trên máy bị quản lý), dùng YAML để mô tả trạng thái mong muốn.

**2. Control Node và Managed Node khác nhau như thế nào?**

> Đáp án: Control Node là máy duy nhất cần cài Ansible, nơi chạy lệnh và lưu inventory/playbook. Managed Node là máy bị quản lý, không cần cài Ansible, chỉ cần SSH server và Python — Ansible đẩy module tới thực thi tại chỗ rồi xóa đi.

**3. Inventory là gì? Kể tên 2 định dạng file inventory tĩnh phổ biến.**

> Đáp án: Inventory là danh sách host và cách nhóm host mà Ansible dùng để biết quản lý máy nào. Hai định dạng tĩnh phổ biến: INI và YAML.

**4. Sự khác nhau giữa Variables và Facts?**

> Đáp án: Variables là giá trị do người dùng chủ động khai báo (tên gói, cổng dịch vụ...). Facts là dữ liệu Ansible tự động thu thập từ managed node (hostname, IP, hệ điều hành...) thông qua module `setup`.

**5. Ad-hoc command dùng để làm gì? Cho một ví dụ.**

> Đáp án: Dùng để gọi một module Ansible trực tiếp, một lần, không cần viết file Playbook — phù hợp cho tác vụ kiểm tra nhanh hoặc khẩn cấp. Ví dụ: `ansible webservers -m ansible.builtin.ping`.

## Mid-level

**6. Vì sao Ansible được gọi là "agentless"? Điều này ảnh hưởng gì tới việc onboard một server mới vào hệ thống automation?**

> Đáp án: Vì managed node không cần cài phần mềm agent thường trực — chỉ cần SSH server (thường có sẵn) và Python. Nhờ vậy, onboard một server mới chỉ cần thêm nó vào inventory và đảm bảo SSH key/sudo đã cấu hình, không cần bước "đăng ký" agent với master như mô hình Puppet/Chef — giảm đáng kể thời gian và điểm lỗi khi mở rộng hạ tầng.

**7. Giải thích thứ tự ưu tiên đọc `ansible.cfg`. Vì sao best practice là đặt file này trong thư mục dự án thay vì `/etc/ansible/ansible.cfg`?**

> Đáp án: Thứ tự: biến môi trường `ANSIBLE_CONFIG` > `./ansible.cfg` (thư mục hiện tại) > `~/.ansible.cfg` > `/etc/ansible/ansible.cfg`, dừng ở file đầu tiên tìm thấy. Đặt trong thư mục dự án và commit Git đảm bảo mọi kỹ sư và CI/CD runner dùng chung đúng một cấu hình, tránh phụ thuộc vào cấu hình máy cá nhân hoặc hệ thống — nhất quán giữa các môi trường.

**8. Tại sao nên ưu tiên module chuyên biệt (`apt`, `user`, `service`) hơn `command`/`shell` khi có thể?**

> Đáp án: Module chuyên biệt tự kiểm tra trạng thái hiện tại trước khi hành động (idempotent), trả về `changed: false` nếu trạng thái đã đúng mong muốn. `command`/`shell` chạy lệnh vô điều kiện mỗi lần, không có khái niệm idempotency built-in, dễ gây side-effect khi chạy lại nhiều lần và khó xác định chính xác điều gì đã thay đổi.

**9. Dynamic Inventory giải quyết vấn đề gì mà Static Inventory không giải quyết được?**

> Đáp án: Với hạ tầng thay đổi liên tục (auto-scaling, container ngắn hạn, cloud instance tạo/xóa động), duy trì danh sách host thủ công trong file tĩnh là bất khả thi và dễ lỗi thời. Dynamic Inventory (script hoặc plugin) truy vấn nguồn dữ liệu thật (API cloud provider, CMDB) tại thời điểm chạy, luôn phản ánh đúng trạng thái hạ tầng hiện tại.

## Thực chiến

**10. Một playbook chạy thành công trên `web-01` nhưng báo `UNREACHABLE` trên `web-02`, dù cả hai đều nằm trong cùng nhóm `webservers` với cùng cấu hình SSH key. Bạn debug theo trình tự nào?**

> Đáp án: (1) Test SSH thủ công trực tiếp tới `web-02` để tách biệt lỗi hạ tầng khỏi lỗi Ansible. (2) Nếu SSH thủ công OK, chạy `ansible-inventory --host web-02` để xác nhận biến kết nối (`ansible_host`, `ansible_user`) không bị override sai ở đâu đó (có thể `host_vars/web-02.yml` có giá trị sai riêng cho host này). (3) Chạy `ansible web-02 -m ping -vvv` để xem chi tiết lệnh SSH thực tế Ansible gọi, so sánh với lệnh SSH thủ công đã thành công ở bước 1. (4) Kiểm tra `known_hosts` — nếu `web-02` là IP mới join gần đây, có thể chưa có fingerprint.

**11. Bạn được giao thiết kế inventory cho hạ tầng hybrid: 50 server on-premise cố định (static) và một Auto Scaling Group trên AWS có thể co giãn từ 2-20 instance. Bạn sẽ tổ chức inventory như thế nào?**

> Đáp án: Dùng inventory tĩnh (YAML/INI) cho 50 server on-premise vì chúng ổn định, thay đổi hiếm. Dùng inventory plugin `amazon.aws.aws_ec2` cho nhóm trên AWS, lọc theo tag (ví dụ `Environment: production`, `Role: webserver`) để tự động nhóm instance mới join mà không cần sửa file thủ công. Ansible hỗ trợ khai báo **nhiều nguồn inventory cùng lúc** (một thư mục chứa cả file tĩnh và file plugin, hoặc dùng tham số `-i` nhiều lần) — kết quả được gộp lại thành một inventory thống nhất.

**12. Sếp yêu cầu bạn giải thích ngắn gọn (30 giây) cho một PM không rành kỹ thuật: "Tại sao chạy Ansible 2 lần liên tiếp lại an toàn, không giống chạy một bash script tự chế 2 lần?"**

> Đáp án mẫu: "Ansible được thiết kế theo nguyên tắc idempotent — mỗi bước tự kiểm tra 'việc này đã xong chưa' trước khi làm, nếu đã xong thì bỏ qua, không làm lại. Một bash script tự viết thường không có bước kiểm tra này, nên chạy lần 2 có thể tạo user trùng, ghi file đè không cần thiết, hoặc gây lỗi không lường trước. Đó là lý do team hạ tầng tin tưởng chạy lại Ansible bất cứ lúc nào để đảm bảo hệ thống đúng trạng thái mong muốn, mà không lo phá vỡ gì thêm."
