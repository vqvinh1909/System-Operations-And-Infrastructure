---
title: "07 - Interview Questions"
module: 20
tags: [docker, ansible, sysops-infra, module-20, capstone, interview]
---

# 07 — Câu hỏi phỏng vấn

## Câu hỏi Junior

1. **Vì sao cần tách HAProxy, Web server, và Database ra các VM riêng biệt thay vì chạy chung một máy?**
   *Đáp án tham khảo:* Tránh single point of failure, mô phỏng đúng kiến trúc 3 tầng phổ biến trong doanh nghiệp (bảo mật — database không public trực tiếp; vận hành — mỗi tầng scale độc lập theo nhu cầu riêng).

2. **Vì sao Ansible dùng SSH key thay vì mật khẩu để kết nối tới các managed node?**
   *Đáp án tham khảo:* An toàn hơn (không truyền mật khẩu qua mạng, không cần lưu mật khẩu ở đâu đó để tự động hóa), và bắt buộc để chạy tự động không cần nhập thủ công mỗi lần — nền tảng cho toàn bộ automation.

3. **Round-robin và leastconn trong HAProxy khác nhau ở điểm nào?**
   *Đáp án tham khảo:* Round-robin phân phối request lần lượt đều nhau giữa các backend. Leastconn ưu tiên gửi request tới backend đang có ít kết nối đang xử lý nhất, phù hợp hơn khi thời gian xử lý request không đồng đều giữa các lần.

4. **Vì sao Web01 và Web02 cần "stateless", và Redis giúp gì cho việc đó?**
   *Đáp án tham khảo:* Vì Load Balancer phân phối request luân phiên giữa hai node, nếu mỗi node tự lưu session riêng, người dùng sẽ mất trạng thái khi request rơi vào node khác. Redis làm nơi lưu session tập trung, cả hai node cùng đọc/ghi vào đó thay vì lưu cục bộ.

5. **RPO và RTO là gì?**
   *Đáp án tham khảo:* RPO (Recovery Point Objective) là lượng dữ liệu tối đa chấp nhận mất tính theo thời gian nếu sự cố xảy ra ngay trước lần backup tiếp theo. RTO (Recovery Time Objective) là thời gian tối đa chấp nhận để khôi phục lại dịch vụ sau sự cố.

## Câu hỏi Mid-level

6. **Trình bày vì sao rsync và volume backup phục vụ hai mục đích khác nhau, không thay thế nhau trong cùng một dự án.**
   *Đáp án tham khảo:* rsync phù hợp cho dữ liệu dạng file (mã nguồn, cấu hình tĩnh), hỗ trợ đồng bộ tăng dần hiệu quả. Volume backup đóng gói toàn bộ trạng thái một Docker volume tại một thời điểm, phù hợp cho dữ liệu database nơi tính toàn vẹn của toàn bộ tập dữ liệu tại một thời điểm quan trọng hơn việc đồng bộ từng file riêng lẻ.

7. **Vì sao cấu hình HAProxy nên được template hóa bằng Jinja2 đọc từ inventory, thay vì viết cứng danh sách IP backend trực tiếp?**
   *Đáp án tham khảo:* Để thêm/bớt Web server trong tương lai chỉ cần sửa inventory và chạy lại playbook, không cần sửa tay file cấu hình HAProxy — đảm bảo tính nhất quán và giảm rủi ro lỗi thao tác thủ công, đúng triết lý Infrastructure as Code.

8. **Giải thích nguyên tắc idempotent trong Ansible, và vì sao nó đặc biệt quan trọng khi playbook được chạy lại nhiều lần trên hạ tầng đang chạy production (không phải VM sạch).**
   *Đáp án tham khảo:* Idempotent nghĩa là chạy playbook nhiều lần trên cùng trạng thái sẽ luôn cho ra cùng kết quả, không gây thay đổi thêm nếu trạng thái đã đúng như mong muốn. Quan trọng vì trên production, việc chạy lại playbook (để cập nhật một phần nhỏ) không được phép gây gián đoạn những phần không thay đổi — nếu không idempotent, mỗi lần chạy lại có thể vô tình restart service không cần thiết hoặc ghi đè cấu hình đang chạy tốt.

9. **Vì sao Prometheus cần một VM/host riêng để giám sát tập trung thay vì mỗi node tự chạy Prometheus riêng cho chính nó?**
   *Đáp án tham khảo:* Giám sát tập trung cho phép nhìn toàn cảnh hạ tầng từ một nơi duy nhất (một dashboard, một nơi quản lý alert rule), và tránh việc Prometheus (một service tốn tài nguyên) cạnh tranh tài nguyên với chính service đang được nó giám sát trên cùng một node.

## Câu hỏi thực chiến

10. **Bạn được giao dự án này trong thực tế với ngân sách chỉ đủ 3 VM thay vì 5. Bạn sẽ thiết kế lại kiến trúc thế nào, đánh đổi gì?**
    *Đáp án tham khảo:* Có thể gộp HAProxy vào chung VM với một trong hai Web server (chấp nhận rủi ro nhỏ hơn vì HAProxy và một web server chia sẻ tài nguyên, nhưng vẫn giữ được 2 web server độc lập cho tính sẵn sàng cao ở tầng ứng dụng), hoặc gộp Control Node vào chung VM Database (vì Control Node không cần chạy liên tục, chỉ cần lúc triển khai). Đánh đổi chính: giảm mức độ cô lập giữa các vai trò, tăng rủi ro một VM chết ảnh hưởng nhiều vai trò cùng lúc — cần ghi rõ đánh đổi này khi trình bày với người ra quyết định ngân sách, không âm thầm chấp nhận rủi ro mà không ai biết.

11. **Trong lúc trình bày dự án này ở buổi phỏng vấn, người phỏng vấn hỏi "nếu VM Database chết hoàn toàn ngay bây giờ, hệ thống của bạn mất bao lâu để khôi phục, và mất bao nhiêu dữ liệu?" Bạn trả lời dựa trên cơ sở nào, không phải đoán?**
    *Đáp án tham khảo:* Trả lời dựa trên RPO/RTO đã tự đặt ra và **đã kiểm chứng thật** qua diễn tập khôi phục (disaster recovery drill) chứ không chỉ trên lý thuyết — ví dụ "RPO tối đa 24h vì backup chạy mỗi ngày 1 lần, RTO khoảng 30 phút vì đã từng thực hành khôi phục từ file backup và đo thời gian thật". Câu trả lời thuyết phục nhất trong phỏng vấn luôn là câu trả lời dựa trên việc đã thực sự làm và đo lường, không phải suy luận trên giấy.

12. **Bạn thiết kế lại toàn bộ dự án này để chuẩn bị chuyển sang chạy trên Kubernetes thay vì Docker Compose thuần túy trên từng VM. Những thành phần nào trong kiến trúc hiện tại sẽ thay đổi vai trò/biến mất, những thành phần nào về cơ bản vẫn giữ nguyên khái niệm?**
    *Đáp án tham khảo:* HAProxy (Load Balancer thủ công) có thể được thay thế bằng Ingress Controller/Service của Kubernetes — khái niệm load balancing/health check vẫn giữ nguyên, chỉ đổi công cụ triển khai. Ansible để cấu hình từng VM sẽ chuyển vai trò sang chủ yếu quản lý hạ tầng nền (provisioning VM, cài Kubernetes) thay vì tự tay deploy từng Docker Compose stack — việc deploy ứng dụng chuyển sang manifest/Helm chart của Kubernetes. Prometheus/Grafana về cơ bản giữ nguyên vai trò và thường được tích hợp sâu hơn nữa với Kubernetes qua các exporter chuyên dụng. Database (MySQL/Redis/RabbitMQ) thường vẫn được khuyến nghị vận hành ngoài Kubernetes hoặc dùng Operator chuyên biệt, vì trạng thái (stateful) trong Kubernetes phức tạp hơn đáng kể so với ứng dụng stateless — đây chính là một trong những chủ đề trọng tâm của khóa Kubernetes tiếp theo.
