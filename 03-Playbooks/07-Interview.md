---
title: "07 - Interview"
module: 3
tags: [ansible, sysops-infra, module-03, interview]
---

# 07 — Câu hỏi phỏng vấn

## Junior

**1. Playbook là gì? Playbook khác Ad-hoc command ở điểm nào?**

> Đáp án: Playbook là file YAML mô tả một chuỗi Task có thứ tự, có thể lưu Git, review, tái sử dụng, chạy lại nhất quán. Ad-hoc command chỉ gọi một module một lần, không lưu lại, phù hợp cho kiểm tra/tác vụ khẩn cấp, không phù hợp cho quy trình nhiều bước.

**2. `loop` dùng để làm gì? Biến đặc biệt nào đại diện cho từng phần tử trong `loop`?**

> Đáp án: Dùng để lặp một Task trên danh sách giá trị, tránh viết lặp lại nhiều Task giống nhau. Biến đặc biệt là `item`.

**3. `when` dùng để làm gì?**

> Đáp án: Chạy Task có điều kiện, dựa trên giá trị Variables hoặc Facts — ví dụ chỉ chạy Task trên hệ điều hành cụ thể.

**4. Handler khác Task thông thường ở điểm nào?**

> Đáp án: Handler chỉ chạy khi được một Task khác `notify` và Task đó thực sự báo `changed: true`. Theo mặc định, Handler chạy ở cuối Play, không chạy ngay tại vị trí notify.

**5. `ignore_errors: true` làm gì?**

> Đáp án: Cho phép Playbook tiếp tục chạy các Task tiếp theo dù Task hiện tại thất bại, thay vì dừng lại theo mặc định.

## Mid-level

**6. Giải thích chi tiết cơ chế Handler: vì sao 3 Task cùng `notify` một Handler thì Handler chỉ chạy đúng một lần?**

> Đáp án: Ansible dùng cơ chế hàng đợi nội bộ (notification queue) — khi một Task báo `changed` và có `notify`, tên Handler được thêm vào hàng đợi, nhưng hàng đợi tự loại trùng tên. Tới cuối Play (hoặc khi gặp `meta: flush_handlers`), Ansible chạy từng Handler duy nhất trong hàng đợi một lần, theo đúng thứ tự Handler được khai báo trong `handlers:` (không phải thứ tự notify).

**7. So sánh `ignore_errors` và `failed_when`. Cho một tình huống nên dùng `failed_when` thay vì để mặc định.**

> Đáp án: `ignore_errors` bỏ qua lỗi Ansible tự phát hiện (thường dựa vào exit code khác 0). `failed_when` cho phép định nghĩa lại logic "thế nào là lỗi" dựa trên nội dung output, kể cả khi exit code là 0. Tình huống: chạy script healthcheck trả về `rc=0` (thành công theo hệ điều hành) nhưng nội dung output chứa `"STATUS: DEGRADED"` — cần `failed_when: "'DEGRADED' in result.stdout"` để Ansible coi đây là lỗi thực sự về mặt nghiệp vụ.

**8. `block/rescue/always` giải quyết vấn đề gì mà `ignore_errors` không giải quyết được?**

> Đáp án: `ignore_errors` chỉ đơn thuần bỏ qua lỗi và tiếp tục — không có cơ chế phản ứng lại lỗi (như rollback), và không đảm bảo một hành động luôn chạy bất kể kết quả. `block/rescue/always` cho phép: nhóm nhiều Task chung một điều kiện xử lý lỗi, chạy logic khắc phục cụ thể khi có lỗi (`rescue`), và đảm bảo một số hành động luôn thực thi dù thành công hay thất bại (`always`) — mô hình try/except/finally đầy đủ hơn nhiều so với việc chỉ bỏ qua lỗi.

**9. Vì sao `import_tasks` không phù hợp khi tên file phụ thuộc vào một biến chỉ có giá trị lúc runtime, còn `include_tasks` thì phù hợp?**

> Đáp án: `import_tasks` được Ansible xử lý (resolve) ngay lúc **parse** toàn bộ Playbook, trước khi kết nối managed node — tại thời điểm đó các biến phụ thuộc facts/runtime chưa có giá trị. `include_tasks` được xử lý lúc **chạy tới** dòng include đó, khi mọi biến/facts đã có giá trị thật, nên hỗ trợ tên file động đầy đủ.

## Thực chiến

**10. Bạn triển khai một bản cập nhật cấu hình nginx trên 20 web server qua Playbook, dùng `--limit`. Sau khi chạy, bạn phát hiện 18 server thành công nhưng 2 server báo lỗi giữa chừng — nginx trên 2 server đó hiện đang ở trạng thái dừng do Task restart thất bại. Bạn xử lý tình huống này thế nào, và thiết kế lại Playbook ra sao để lần sau không xảy ra rủi ro tương tự?**

> Đáp án: Trước mắt, xử lý khẩn cấp: SSH thủ công hoặc chạy ad-hoc restart nginx trên 2 server bị ảnh hưởng để khôi phục dịch vụ ngay. Sau đó điều tra log Ansible (`-vvv`) để tìm nguyên nhân gốc của lỗi restart. Thiết kế lại: bọc chuỗi "copy config + restart" trong `block`, thêm `rescue` để tự động rollback về file cấu hình cũ (backup trước khi ghi đè bằng `backup: true` của module `copy`/`template`) nếu restart thất bại, và `always` đảm bảo trạng thái service được kiểm tra lại (`systemd: state=started`) dù nhánh nào xảy ra — tránh để service ở trạng thái dừng dở dang như tình huống vừa gặp.

**11. Một Playbook cũ trong team dùng rất nhiều `command`/`shell` kèm `ignore_errors: true` ở hầu hết Task, và không dùng `register`/`when` để kiểm tra kết quả. Bạn được giao refactor lại. Bạn ưu tiên sửa gì trước, theo thứ tự nào, và vì sao?**

> Đáp án: (1) Trước tiên, rà soát từng `ignore_errors: true` — phân loại Task nào thực sự không quan trọng (giữ nguyên) và Task nào đang che giấu lỗi thật (bỏ `ignore_errors`, xử lý đúng bằng `block/rescue` hoặc `failed_when`). Đây là ưu tiên cao nhất vì lỗi bị che giấu là rủi ro production nghiêm trọng nhất. (2) Thay `command`/`shell` bằng module chuyên biệt tương ứng khi có thể, khôi phục idempotency thật sự. (3) Với `command`/`shell` không thể thay thế (không có module tương đương), thêm `register` + `changed_when`/`failed_when` tường minh để Ansible báo cáo đúng trạng thái thay vì luôn coi là `changed`. (4) Cuối cùng mới tối ưu cấu trúc (tách file, thêm tags) — vì đây là cải thiện khả năng bảo trì, ít khẩn cấp hơn rủi ro về đúng-sai logic ở bước 1-3.

**12. Sếp hỏi: "Tại sao Handler restart service ở cuối Play mà không restart ngay khi vừa sửa xong file cấu hình? Chẳng phải như vậy có độ trễ giữa lúc sửa và lúc áp dụng không?"**

> Đáp án mẫu: "Đúng là có độ trễ, nhưng đó là thiết kế có chủ đích. Nếu Play có 5 Task cùng sửa 5 file cấu hình khác nhau của cùng một service, restart ngay sau mỗi Task nghĩa là service bị restart tới 5 lần trong một lần chạy — gây gián đoạn không cần thiết và chậm hơn. Bằng cách gom lại và chỉ restart một lần ở cuối, sau khi mọi thay đổi cấu hình đã áp dụng xong, ta vừa giảm downtime vừa đảm bảo service khởi động lại với cấu hình cuối cùng hoàn chỉnh, không phải trạng thái dở dang giữa chừng."
