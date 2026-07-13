---
title: "07 - Interview Questions"
module: 0
tags: [ansible, sysops-infra, module-00, interview]
---

# 07 — Câu hỏi phỏng vấn

## Junior

**Infrastructure as Code là gì? Cho ví dụ thực tế.**
IaC là cách quản lý và cấp phát hạ tầng bằng file code (YAML, HCL...) thay vì thao tác thủ công. Ví dụ: thay vì SSH vào từng server để cài Nginx, viết một Ansible playbook khai báo "Nginx phải được cài, service phải chạy", rồi áp dụng playbook đó lên toàn bộ server cùng lúc.

**Idempotency là gì? Vì sao quan trọng trong automation?**
Là tính chất một thao tác cho ra cùng kết quả dù chạy 1 lần hay nhiều lần, và không gây tác dụng phụ khi chạy lại lúc trạng thái đã đúng. Quan trọng vì nó cho phép chạy lại playbook an toàn sau khi lỗi giữa chừng, và dùng playbook để vừa provision vừa kiểm tra compliance định kỳ mà không sợ phá vỡ hệ thống đang chạy đúng.

**Nêu 2 nhược điểm của quản trị hạ tầng thủ công (Traditional).**
(1) Configuration drift — cấu hình dần lệch khỏi chuẩn theo thời gian do sửa tay không ghi lại; (2) không scale được — cấu hình thủ công 100 server tốn quá nhiều thời gian và dễ sai sót con người so với chạy một lệnh tự động.

**Ansible là Agent-based hay Agentless? Giải thích ngắn gọn.**
Agentless. Ansible không cần cài phần mềm agent thường trực trên managed node — chỉ cần SSH server đang chạy và Python có sẵn. Control node kết nối qua SSH, đẩy (push) module cần thiết, thực thi, rồi dọn dẹp.

## Mid-level

**Phân biệt Declarative và Imperative. Ansible thuộc loại nào?**
Imperative mô tả từng bước cụ thể cần làm theo thứ tự; công cụ thực thi đúng như viết mà không tự kiểm tra trạng thái hiện tại. Declarative mô tả trạng thái mong muốn cuối cùng; công cụ tự so sánh và quyết định cần làm gì để đạt trạng thái đó. Ansible là declarative ở cấp độ từng task (module tự quyết định hành động dựa trên trạng thái hiện tại), nhưng playbook thực thi tuần tự theo thứ tự viết — mang tính imperative ở cấp độ playbook.

**So sánh Agent-based (Puppet/Chef) và Agentless (Ansible) — nêu ít nhất 2 đánh đổi.**
Agent-based: cần cài và duy trì agent trên từng node, tiêu tốn tài nguyên thường trực, nhưng có lợi thế khi scale tới hàng chục nghìn node nhờ mô hình pull tự động theo chu kỳ. Agentless: không cần cài gì thêm ngoài SSH/Python có sẵn, áp dụng thay đổi tức thời (push model), tận dụng lại hạ tầng bảo mật SSH đã có, nhưng cần thêm kiến trúc riêng (execution strategy, Automation Platform) để scale tốt ở quy mô rất lớn.

**Configuration drift là gì? Làm sao IaC giúp phát hiện và khắc phục?**
Là hiện tượng cấu hình thực tế của server lệch khỏi trạng thái chuẩn đã khai báo, thường do sửa tay ngoài quy trình. IaC giúp phát hiện bằng cách chạy lại playbook ở chế độ dry-run (`--check`) định kỳ — nếu Ansible báo `changed` nghĩa là có lệch so với khai báo. Khắc phục bằng cách chạy lại playbook thật (không sửa tay lại) để đưa server về đúng trạng thái khai báo trong code.

**Mutable Infrastructure và Immutable Infrastructure khác nhau ở điểm nào? Ansible phù hợp mô hình nào hơn?**
Mutable: server tồn tại lâu dài, được sửa tại chỗ mỗi khi cập nhật. Immutable: server không bao giờ sửa sau khi triển khai — cần thay đổi thì build và triển khai server/image mới rồi tiêu hủy server cũ. Ansible vận hành tốt nhất với mô hình Mutable, vì nó được thiết kế để SSH vào và chỉnh sửa server đang sống lặp đi lặp lại; container/Kubernetes thiên về Immutable.

## Thực chiến (Scenario-based)

**Bạn phát hiện một server production có cấu hình firewall khác các server còn lại dù chạy chung một playbook. Bạn sẽ điều tra và xử lý thế nào?**
Trước tiên chạy playbook với `--check --diff` nhắm vào server đó để xem Ansible phát hiện gì khác biệt so với khai báo. Nếu có khác biệt, kiểm tra xem đó là do (a) playbook thực sự chưa từng chạy đúng trên server này (kiểm tra log lần chạy trước), hay (b) có ai đó sửa tay ngoài quy trình (configuration drift). Nếu là (b), không sửa tay tiếp — chạy lại playbook thật để đồng bộ, đồng thời báo cáo và siết lại quy trình (branch protection, hạn chế quyền SSH trực tiếp vào production) để tránh tái diễn. Ghi nhận sự việc vào postmortem nếu gây ảnh hưởng.

**Sếp yêu cầu bạn thuyết phục đội ngũ (vốn quen thao tác thủ công nhiều năm) chuyển sang IaC. Họ phản đối vì "thao tác tay nhanh hơn, học công cụ mới mất thời gian". Bạn thuyết phục thế nào?**
Không phủ nhận thao tác tay nhanh hơn cho **1 server, 1 lần**. Nhưng đưa ra ví dụ cụ thể: vá lỗ hổng bảo mật khẩn cấp trên 50 server — thao tác tay mất nhiều giờ và rủi ro bỏ sót/gõ sai; với Ansible, một lệnh áp dụng đồng loạt trong vài phút, có log đầy đủ ai chạy khi nào. Đề xuất bắt đầu nhỏ: áp dụng IaC cho một tác vụ lặp lại thường xuyên và ít rủi ro trước (ví dụ đồng bộ file cấu hình NTP, tạo user chuẩn) để đội thấy giá trị thực tế trước khi mở rộng ra toàn bộ hạ tầng — giảm nỗi sợ thay đổi bằng kết quả cụ thể thay vì lý thuyết.

**Một đồng nghiệp nói "vì Ansible playbook chạy tuần tự từ trên xuống nên nó không phải công cụ declarative thật sự — đây là điểm yếu". Bạn phản biện thế nào?**
Đây là hiểu chưa đầy đủ về khái niệm declarative trong ngữ cảnh Ansible. Ansible declarative ở **cấp độ module/task** — mỗi task khai báo trạng thái mong muốn và module tự quyết định hành động dựa trên trạng thái hiện tại, không phải danh sách lệnh cứng. Việc playbook chạy tuần tự là đặc điểm ở **cấp độ điều phối** (orchestration), giúp người viết kiểm soát rõ ràng thứ tự phụ thuộc giữa các bước (ví dụ phải cài package trước khi cấu hình service) — đây là điểm mạnh chứ không phải điểm yếu, vì hạ tầng thực tế luôn có phụ thuộc thứ tự thật (ví dụ không thể start service trước khi cài xong package).
