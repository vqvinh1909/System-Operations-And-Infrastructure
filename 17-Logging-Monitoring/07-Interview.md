---
title: "07 - Interview Questions"
module: 17
tags: [docker, sysops-infra, module-17, logging, monitoring, interview]
---

# 07 — Câu hỏi phỏng vấn

## Câu hỏi Junior

1. **Log driver mặc định của Docker là gì, và rủi ro lớn nhất nếu không cấu hình gì thêm là gì?**
   *Đáp án tham khảo:* `json-file`. Rủi ro lớn nhất là không có giới hạn dung lượng mặc định — một container log liên tục có thể làm đầy toàn bộ đĩa `/var/lib/docker`, ảnh hưởng tới mọi container khác trên cùng host.

2. **Vì sao log của một container có thể biến mất hoàn toàn dù trước đó `docker logs` vẫn xem được bình thường?**
   *Đáp án tham khảo:* Log driver `json-file` lưu log gắn liền với container ID cụ thể. Khi container bị xóa (không chỉ restart mà bị recreate, ví dụ do deploy bản mới), file log tương ứng cũng bị xóa theo, trừ khi có hệ thống thu thập log tập trung (Loki, ELK...) đã sao chép dữ liệu ra ngoài trước đó.

3. **Prometheus thu thập dữ liệu theo mô hình pull hay push? Giải thích ngắn gọn.**
   *Đáp án tham khảo:* Pull. Prometheus chủ động gửi HTTP request tới endpoint `/metrics` của từng target theo chu kỳ cố định (scrape interval), thay vì chờ ứng dụng tự đẩy dữ liệu tới.

4. **cAdvisor dùng để làm gì?**
   *Đáp án tham khảo:* Thu thập thông tin tài nguyên (CPU, RAM, network, disk I/O) của từng container riêng lẻ từ cgroup của kernel, rồi expose dữ liệu đó dưới dạng endpoint đúng chuẩn Prometheus để Prometheus scrape.

5. **Grafana có tự lưu trữ dữ liệu metric/log không?**
   *Đáp án tham khảo:* Không. Grafana chỉ là lớp truy vấn và trực quan hóa, kết nối tới các datasource bên ngoài như Prometheus (metric) và Loki (log) để lấy dữ liệu hiển thị.

## Câu hỏi Mid-level

6. **Loki khác Elasticsearch (ELK stack) ở điểm thiết kế cốt lõi nào?**
   *Đáp án tham khảo:* Loki chỉ index một tập nhỏ label gắn với mỗi luồng log, không index toàn bộ nội dung (full-text) như Elasticsearch. Nội dung log được nén thành chunk và chỉ quét khi truy vấn đã lọc theo label trước. Điều này giúp Loki rẻ hơn đáng kể ở cùng khối lượng dữ liệu, nhưng bắt buộc truy vấn phải luôn bắt đầu bằng bộ lọc label.

7. **Vì sao Promtail không còn được khuyến nghị dùng cho triển khai mới tính đến giữa năm 2026?**
   *Đáp án tham khảo:* Promtail đã chuyển sang giai đoạn Long-Term Support từ 13/02/2025 và chính thức End-Of-Life ngày 02/03/2026 — không còn nhận bản vá bảo mật, không phát triển tính năng mới. Grafana Labs khuyến nghị chuyển sang Grafana Alloy, agent thế hệ mới xây trên nền OpenTelemetry Collector.

8. **Trong sơ đồ pipeline Metrics, mũi tên giữa cAdvisor và Prometheus vẽ theo hướng nào, và vì sao dễ gây hiểu nhầm?**
   *Đáp án tham khảo:* Sơ đồ thường vẽ mũi tên từ cAdvisor sang Prometheus theo hướng "dữ liệu chảy về", nhưng thực tế bên chủ động khởi tạo kết nối TCP lại là Prometheus (gửi HTTP GET tới cAdvisor theo mô hình pull). Cần phân biệt rõ hướng luồng dữ liệu logic và hướng khởi tạo kết nối mạng thực tế khi troubleshoot network/firewall.

9. **Vì sao cần cả hai: giám sát host-level (Zabbix) và giám sát container-native (stack module này) trong cùng một hạ tầng doanh nghiệp?**
   *Đáp án tham khảo:* Zabbix trả lời câu hỏi ở mức toàn máy (host còn sống không, CPU/RAM/disk tổng thể, dịch vụ hệ thống có chạy không) — cần thiết để biết hạ tầng nền tảng ổn định. Stack container-native trả lời câu hỏi ở mức từng container riêng lẻ, thứ Zabbix không có khả năng phân giải khi một host chạy hàng chục container liên tục sinh ra/mất đi. Hai tầng bổ sung, không thay thế nhau.

## Câu hỏi thực chiến

10. **Dashboard Grafana của bạn đột nhiên không còn dữ liệu mới từ 2 giờ trước, dù Grafana UI vẫn truy cập bình thường. Bạn chẩn đoán theo thứ tự nào?**
    *Đáp án tham khảo:* Đầu tiên kiểm tra Grafana có kết nối được datasource không (Test datasource) để loại trừ lỗi mạng Grafana-Prometheus. Sau đó kiểm tra trạng thái target trên chính Prometheus UI (`/targets`) — nếu target `DOWN` từ đúng 2 giờ trước, khả năng cao là container đang bị scrape (cAdvisor) đã crash hoặc restart mất kết nối mạng. Kiểm tra `docker ps`/`docker logs` của container đó để xác nhận. Cuối cùng mới nghi ngờ tới chính Prometheus (đĩa đầy khiến TSDB ngừng ghi, hoặc container Prometheus tự restart) nếu các bước trên đều bình thường.

11. **Một container production log ra thông tin nhạy cảm (ví dụ log nhầm mật khẩu database trong lúc debug) và dữ liệu đó đã được Alloy đẩy vào Loki. Bạn xử lý sự cố này như thế nào, cả về mặt kỹ thuật lẫn quy trình?**
    *Đáp án tham khảo:* Về kỹ thuật: Loki không hỗ trợ xóa từng dòng log dễ dàng như xóa một bản ghi database — cần đánh giá dùng tính năng xóa theo compactor/retention hoặc xóa toàn bộ chunk chứa dữ liệu đó nếu Loki hỗ trợ (tùy version/cấu hình lưu trữ), việc này ảnh hưởng cả các log hợp lệ khác trong cùng chunk. Về quy trình: đây là sự cố rò rỉ dữ liệu nhạy cảm — cần báo ngay theo quy trình bảo mật nội bộ (không tự ý xử lý âm thầm), đổi ngay mật khẩu đã bị lộ, và quan trọng nhất là sửa gốc rễ (code đang log nhầm thông tin nhạy cảm) để không lặp lại, đồng thời rà soát xem log đó đã bị ai truy cập qua Grafana chưa.

12. **Bạn được giao thiết kế retention policy (thời gian giữ dữ liệu) cho Prometheus và Loki trong một hạ tầng có ngân sách hạn chế. Bạn sẽ cân nhắc những yếu tố gì để quyết định giữ bao lâu?**
    *Đáp án tham khảo:* Cân đối giữa nhu cầu điều tra sự cố (thường sự cố được phát hiện và điều tra trong vài ngày tới vài tuần) với chi phí lưu trữ (dung lượng đĩa tăng tuyến tính theo thời gian giữ và khối lượng dữ liệu). Thực hành phổ biến: giữ dữ liệu độ phân giải cao (raw) trong thời gian ngắn hơn (ví dụ 15-30 ngày) cho cả metric lẫn log, với metric có thể cân nhắc thêm downsampling (giảm độ chi tiết) cho dữ liệu cũ hơn nếu cần lưu dài hạn phục vụ báo cáo xu hướng. Cũng cần đối chiếu với yêu cầu compliance của tổ chức (nếu có) — một số ngành yêu cầu giữ log tối thiểu một khoảng thời gian nhất định theo luật.
