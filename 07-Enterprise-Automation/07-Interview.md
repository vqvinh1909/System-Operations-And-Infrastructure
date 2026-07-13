---
title: "07 - Interview"
module: 7
tags: [ansible, sysops-infra, module-07, interview]
---

# 07 — Câu hỏi phỏng vấn

## Junior

**1. `serial` trong Playbook dùng để làm gì?**

> Đáp án: Giới hạn số lượng host được xử lý trong mỗi đợt (batch) của một Play, thay vì Ansible chạy song song toàn bộ host cùng lúc — nền tảng cho Rolling Update.

**2. Zero Downtime Deployment là gì, ở mức khái niệm?**

> Đáp án: Triển khai thay đổi lên hạ tầng đang phục vụ người dùng mà không gây gián đoạn dịch vụ, thường bằng cách phối hợp cập nhật từng phần nhỏ (Rolling Update) với việc tạm gỡ node khỏi Load Balancer trước khi cập nhật.

**3. `max_fail_percentage` dùng để làm gì?**

> Đáp án: Đặt ngưỡng phần trăm host thất bại tối đa trước khi Ansible dừng toàn bộ Playbook, giới hạn phạm vi ảnh hưởng của một lỗi triển khai.

**4. `delegate_to` dùng để làm gì?**

> Đáp án: Chạy một Task cụ thể trên một host khác, thay vì host hiện tại đang được xử lý trong vòng lặp của Play.

**5. Vì sao cần backup database trước khi triển khai thay đổi liên quan schema?**

> Đáp án: Vì thay đổi database (khác Web tier) khó hoặc không thể "rollback" đơn giản bằng cách chạy lại Playbook — cần bản backup để khôi phục nếu thay đổi gây lỗi hoặc mất dữ liệu.

## Mid-level

**6. So sánh `serial: 5` và `serial: "25%"` — khi nào chọn cái nào?**

> Đáp án: `serial: 5` cố định số host mỗi đợt, phù hợp khi muốn kiểm soát chặt số lượng tuyệt đối (ví dụ hạ tầng nhỏ, muốn chắc chắn không quá 5 host bị ảnh hưởng cùng lúc dù tổng số host tăng lên sau này). `serial: "25%"` tự động co giãn theo tổng số host, phù hợp khi hạ tầng thay đổi quy mô thường xuyên và muốn tỷ lệ ảnh hưởng luôn nhất quán tương đối.

**7. Giải thích vì sao chỉ dùng `serial` mà không phối hợp Load Balancer thì chưa đạt Zero Downtime thật sự?**

> Đáp án: `serial` chỉ kiểm soát **tốc độ** Ansible xử lý host, không kiểm soát việc Load Balancer có còn gửi traffic tới host đang bị cập nhật hay không. Nếu không gỡ node khỏi pool trước khi cập nhật, người dùng vẫn có thể bị định tuyến tới một node đang restart service hoặc đang trong trạng thái dở dang, gây lỗi 502/503 dù tổng thời gian gián đoạn ngắn hơn cập nhật toàn bộ cùng lúc.

**8. Vì sao health check "nông" (shallow, chỉ kiểm tra web server có phản hồi) không đủ tin cậy cho Zero Downtime Deployment?**

> Đáp án: Vì web server có thể "đang chạy" nhưng phụ thuộc quan trọng (database, cache, service downstream) chưa sẵn sàng — health check nông không phát hiện được tình trạng này, dẫn tới đưa một node chưa thực sự sẵn sàng trở lại pool, gây lỗi cho người dùng ngay sau khi node được "enable" trở lại.

**9. `run_once: true` kết hợp `delegate_to: localhost` giải quyết vấn đề gì?**

> Đáp án: Đảm bảo một hành động mang tính tổng kết cho toàn bộ Play (ví dụ gửi một thông báo triển khai hoàn tất) chỉ thực thi đúng một lần, trên đúng một máy xác định (thường là control node), thay vì bị lặp lại N lần (một lần cho mỗi host trong Play) hoặc chạy trên một host ngẫu nhiên không mong muốn.

## Thực chiến

**10. Bạn triển khai Rolling Update cho 20 web server với `serial: 4`, `max_fail_percentage: 20`. Ở đợt thứ 3 (host 9-12), 1 host thất bại. Playbook có dừng không? Giải thích cách Ansible tính toán, và bạn sẽ làm gì tiếp theo.**

> Đáp án: `max_fail_percentage` tính trên **tổng số host của toàn Play** (20 host), không phải theo từng đợt riêng lẻ. 1/20 = 5%, thấp hơn ngưỡng 20%, nên Playbook **không dừng**, tiếp tục các đợt còn lại — miễn tổng số host thất bại tích lũy không vượt quá 4/20 (20%). Bước tiếp theo: điều tra ngay nguyên nhân host thất bại ở đợt 3 (không chờ Playbook chạy hết), vì nếu nguyên nhân là lỗi hệ thống (không phải lỗi cá biệt của 1 host), khả năng cao các đợt sau sẽ tiếp tục thất bại và tích lũy tới ngưỡng dừng.

**11. Một đồng nghiệp đề xuất bỏ hoàn toàn bước "chờ drain" (Bước 2 trong chuỗi 6 bước Zero Downtime) để rút ngắn thời gian triển khai, lý do "HAProxy đã disable server rồi thì đâu còn traffic mới nữa". Bạn có đồng ý không? Giải thích.**

> Đáp án: Không nên bỏ hoàn toàn. Đúng là `disable` ngăn traffic **mới** được gửi tới node đó, nhưng các kết nối **đang xử lý dở dang** tại thời điểm disable (ví dụ một request đang chờ database trả kết quả) vẫn cần thời gian để hoàn tất — nếu cập nhật/restart service ngay lập tức, các kết nối dở dang này bị cắt đột ngột, người dùng đang thao tác giữa chừng gặp lỗi. Thời gian chờ drain nên được tính toán dựa trên thời gian xử lý request trung bình/tối đa thực tế của ứng dụng (ví dụ p99 latency), không phải một con số tùy ý — có thể rút ngắn nếu đo đạc thực tế cho thấy phù hợp, nhưng không nên bỏ hoàn toàn.

**12. Sếp yêu cầu bạn thiết kế chiến lược triển khai cho một thay đổi schema database (thêm cột NOT NULL vào bảng 50 triệu dòng) mà không được downtime, trong khi ứng dụng vẫn đang chạy rolling update bình thường. Bạn trình bày cách tiếp cận ở mức cao.**

> Đáp án mẫu: Đây là bài toán cần tách thành nhiều giai đoạn nhỏ tương thích ngược, không thể làm trong một bước: (1) Thêm cột mới cho phép NULL trước (không khóa bảng lâu, không phá vỡ ứng dụng phiên bản cũ đang chạy vì cột NULL không bắt buộc điền). (2) Triển khai (rolling update) phiên bản ứng dụng mới ghi giá trị vào cột mới cho mọi bản ghi mới/cập nhật, đồng thời chạy một job nền backfill dữ liệu cũ theo lô nhỏ (tránh khóa bảng lâu trên 50 triệu dòng). (3) Sau khi xác nhận 100% dữ liệu đã có giá trị ở cột mới, mới thêm ràng buộc NOT NULL thật sự — bước này an toàn vì không còn dòng nào vi phạm ràng buộc. (4) Toàn bộ 3 giai đoạn trên đều tận dụng Ansible để điều phối (Playbook riêng cho từng giai đoạn, backup trước mỗi giai đoạn theo nguyên tắc [[02-Theory|02 - Theory]] mục 6), nhưng logic nghiệp vụ "tương thích ngược" là quyết định kiến trúc ứng dụng, không phải thứ Ansible tự động hóa được — nhấn mạnh rằng Zero Downtime cho database luôn đòi hỏi thiết kế thay đổi kỹ hơn nhiều so với Web tier.
