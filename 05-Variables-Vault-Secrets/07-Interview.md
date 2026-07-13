---
title: "07 - Interview"
module: 5
tags: [ansible, sysops-infra, module-05, interview]
---

# 07 — Câu hỏi phỏng vấn

## Junior

**1. Nếu một biến được khai báo cả trong `group_vars` và `--extra-vars`, giá trị nào được dùng?**

> Đáp án: Giá trị từ `--extra-vars` — đây là nguồn có độ ưu tiên cao nhất trong toàn bộ hệ thống biến của Ansible.

**2. Ansible Vault dùng thuật toán mã hóa nào?**

> Đáp án: AES256.

**3. Lệnh nào dùng để tạo mới một file đã mã hóa ngay từ đầu?**

> Đáp án: `ansible-vault create <file>`.

**4. `ansible-vault view` khác `ansible-vault edit` ở điểm nào?**

> Đáp án: `view` chỉ hiển thị nội dung, không cho sửa và không ghi lại file. `edit` cho phép sửa nội dung, tự động mã hóa lại khi lưu.

**5. `roles/x/defaults/main.yml` có độ ưu tiên biến cao hay thấp so với các nguồn khác?**

> Đáp án: Thấp nhất trong các nguồn biến "thật" — thiết kế để dễ dàng bị ghi đè bởi người dùng Role.

## Mid-level

**6. Giải thích vì sao đặt giá trị mặc định của một Role nên nằm ở `defaults/main.yml`, không nên nằm ở `vars/main.yml`?**

> Đáp án: `defaults` có độ ưu tiên thấp nhất, cho phép người dùng Role dễ dàng ghi đè qua bất kỳ nguồn nào khác (group_vars, host_vars, extra-vars) mà không cần sửa code Role. `vars/main.yml` có độ ưu tiên cao (cấp 15), gần như "cứng", khó bị ghi đè — chỉ nên dùng cho giá trị nội bộ mà Role không muốn/không nên cho người dùng tùy chỉnh.

**7. `encrypt_string` giải quyết vấn đề gì mà `encrypt` (mã hóa cả file) không giải quyết tốt?**

> Đáp án: Mã hóa cả file khiến toàn bộ nội dung — kể cả biến không nhạy cảm — trở nên không thể đọc/review trực tiếp trong Pull Request (reviewer phải giải mã mới xem được diff thật). `encrypt_string` cho phép chỉ mã hóa đúng 1-2 biến nhạy cảm, phần còn lại của file vẫn là plaintext dễ đọc, cân bằng giữa bảo mật và khả năng review code.

**8. Vault ID (`--vault-id ID@SOURCE`) giải quyết bài toán gì trong tổ chức có nhiều môi trường?**

> Đáp án: Cho phép dùng nhiều password Vault khác nhau cho các mục đích/môi trường khác nhau (dev, staging, production) trong cùng một hệ thống Ansible, thay vì bắt buộc dùng chung một password cho mọi secret. Điều này giới hạn phạm vi ảnh hưởng nếu một password bị lộ — người biết password Vault dev không tự động đọc được secret production.

**9. Nếu phát hiện một secret plaintext đã bị commit và push lên Git, bước đầu tiên cần làm là gì — dọn lịch sử Git hay đổi secret thật? Vì sao?**

> Đáp án: Đổi secret thật trước tiên. Vì ngay khi đã push, secret coi như đã bị lộ (không thể chắc chắn không ai đã xem/tải về, kể cả trong khoảng thời gian ngắn) — dọn lịch sử Git chỉ ngăn người xem **sau này**, không thu hồi được việc đã lộ trước đó. Đổi secret thật ngay lập tức là bước duy nhất thực sự vô hiệu hóa rủi ro.

## Thực chiến

**10. Một Playbook chạy đúng trên `web-01` nhưng sai trên `web-02` với cùng một biến `max_connections` — dù cả hai đều thuộc nhóm `webservers` và bạn chắc chắn `group_vars/webservers.yml` chỉ có một giá trị duy nhất. Bạn nghi ngờ điều gì trước tiên, và debug theo trình tự nào?**

> Đáp án: Nghi ngờ đầu tiên: `web-02` có `host_vars/web-02.yml` riêng ghi đè giá trị này (host_vars có độ ưu tiên cao hơn group_vars). Debug: (1) `grep -rn "max_connections" group_vars/ host_vars/` để tìm mọi nơi biến này xuất hiện. (2) `ansible-inventory --host web-02` để xem giá trị cuối cùng đã resolve. (3) Nếu vẫn không rõ, chạy `ansible-playbook -vvvv --check` để xem log chi tiết Ansible đọc biến từ nguồn nào theo thứ tự.

**11. Team bạn đang cân nhắc: (A) mã hóa toàn bộ file `group_vars/production/all.yml` bằng `ansible-vault encrypt`, hay (B) chỉ mã hóa từng biến nhạy cảm bằng `encrypt_string`, giữ file chính ở dạng plaintext. Phân tích ưu nhược điểm mỗi phương án cho một team 8 kỹ sư, review code qua Pull Request là bắt buộc.**

> Đáp án: Phương án (A) đơn giản triển khai, đảm bảo an toàn tuyệt đối cho mọi biến trong file, nhưng làm Pull Request không thể review được nội dung thay đổi thật (reviewer chỉ thấy khối mã hóa thay đổi, không biết giá trị cũ/mới là gì) — làm giảm chất lượng review, dễ bỏ sót lỗi logic không liên quan bảo mật. Phương án (B) giữ được khả năng review đầy đủ cho phần lớn nội dung, chỉ ẩn đúng phần thực sự nhạy cảm — phù hợp hơn cho team ưu tiên quy trình review chặt chẽ, dù tốn công sức thiết lập ban đầu hơn (phải xác định rõ biến nào cần mã hóa). Khuyến nghị thực tế: dùng (B) làm mặc định, chỉ chuyển sang (A) khi số lượng biến nhạy cảm trong file chiếm đa số, khiến việc mã hóa lẻ tẻ trở nên rườm rà hơn lợi ích review mang lại.

**12. Sếp yêu cầu bạn thiết kế quy trình rotate password Vault định kỳ 90 ngày cho môi trường production, đảm bảo không gây gián đoạn CI/CD đang chạy. Bạn thiết kế quy trình này như thế nào ở mức cao?**

> Đáp án mẫu: (1) Tạo password Vault mới, chạy `ansible-vault rekey` trên toàn bộ file Vault production, thực hiện trên một nhánh Git riêng (không merge ngay). (2) Cập nhật password mới vào nơi lưu trữ tập trung mà CI/CD runner đọc (ví dụ biến môi trường được inject qua secret manager của CI/CD, không hard-code trong pipeline config). (3) Merge nhánh chứa file đã rekey và cập nhật secret CI/CD **cùng một thời điểm** (trong cùng một cửa sổ bảo trì ngắn) để tránh tình trạng CI/CD dùng password cũ cố gắng giải mã file đã rekey bằng password mới thất bại. (4) Thông báo trước cho toàn team về thời điểm rotate, vô hiệu hóa password cũ ngay sau khi xác nhận pipeline chạy thành công với password mới. (5) Ghi lại lịch sử rotate (ngày, người thực hiện) phục vụ audit.
