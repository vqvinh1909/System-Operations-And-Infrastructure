---
title: "06 - Troubleshooting (20 Tình huống thực chiến)"
module: 20
tags: [docker, ansible, sysops-infra, module-20, capstone, troubleshooting]
---

# 06 — 20 Tình huống sự cố thực tế trên hạ tầng của bạn

> [!warning] Đây là bài tập tự chẩn đoán, không phải hỏi-đáp
> Mỗi tình huống dưới đây mô tả một bối cảnh và triệu chứng cụ thể xảy ra trên chính hạ tầng 5 VM bạn vừa dựng ở [[05-Labs]]. Không có đáp án đi kèm ngay bên dưới — nhiệm vụ của bạn là áp dụng đúng quy trình debug phân tầng đã học ở [[../19-Troubleshooting/README|Module 19]] để tự chẩn đoán và khắc phục trên chính môi trường lab thật của mình. Mỗi tình huống có một dòng "Tham khảo lại" gợi ý module liên quan nếu bạn cần ôn lại kiến thức nền — đây không phải đáp án, chỉ là điểm khởi đầu để tự tra cứu.

## Nhóm 1 — Sự cố khi triển khai bằng Ansible

**Tình huống 1.** Bạn chạy `ansible-playbook -i inventory.ini site.yml` lần đầu. Playbook chạy thành công trên `haproxy01`, `web01`, `db01`, nhưng dừng lại với lỗi kết nối SSH ngay tại `web02`, dù bạn chắc chắn đã `ssh-copy-id` cho cả 4 node giống hệt nhau.
*Yêu cầu:* xác định chính xác nguyên nhân khiến riêng `web02` không kết nối được, trong khi quy trình bạn thực hiện cho cả 4 node là như nhau.
*Tham khảo lại: Module 02 (Ansible Fundamentals), Module 19 Phần B.*

**Tình huống 2.** Bạn chạy lại `site.yml` lần thứ hai (không thay đổi gì trong code) để kiểm tra tính idempotent. Lần chạy thứ hai này, HAProxy bị mất kết nối tới cả hai Web server trong vài giây, dù không có Web server nào thực sự bị tắt.
*Yêu cầu:* xác định vì sao một playbook chạy lại lần hai (không đổi nội dung) lại gây gián đoạn dịch vụ, trong khi nguyên tắc idempotent lẽ ra phải đảm bảo chạy lại không thay đổi gì nếu trạng thái đã đúng.
*Tham khảo lại: Module 03 (Playbooks), Module 07 (Enterprise Automation).*

**Tình huống 3.** Một thành viên mới trong team được cấp SSH key riêng để quản trị hạ tầng. Sau khi bạn thu hồi (revoke) key SSH cũ của một cựu nhân viên khỏi `authorized_keys` trên cả 4 managed node, Ansible từ Control Node đột nhiên không kết nối được tới bất kỳ node nào nữa, kể cả khi dùng đúng key hiện tại.
*Yêu cầu:* xác định vì sao thao tác thu hồi key của một người không liên quan lại ảnh hưởng tới toàn bộ khả năng kết nối của Control Node.
*Tham khảo lại: Module 05 (Variables, Vault, Secrets), Module 19 Phần B.*

## Nhóm 2 — Sự cố Load Balancer và tầng Web

**Tình huống 4.** HAProxy Stats page cho thấy `web01` ở trạng thái `DOWN` (màu đỏ) dù bạn xác nhận `docker ps` trên chính VM Web01 cho thấy container Nginx vẫn đang `Up` bình thường, và `curl http://localhost:8080` chạy trực tiếp trên Web01 trả về đúng nội dung.
*Yêu cầu:* xác định vì sao HAProxy vẫn coi Web01 là down dù bản thân service trên đó hoạt động bình thường khi kiểm tra cục bộ.
*Tham khảo lại: Module 19 Phần B, [[02-Theory]] mục 2.*

**Tình huống 5.** Người dùng báo cáo thỉnh thoảng nhận lỗi 502 khi truy cập ứng dụng, không phải liên tục mà chỉ khoảng 1 trong 10 lần thử, và không ai trong team nhận thấy bất kỳ node nào bị down khi kiểm tra `docker ps` ngay sau khi nhận báo cáo.
*Yêu cầu:* thiết kế cách bạn sẽ tái tạo lại được hiện tượng "lỗi ngắt quãng, không liên tục" này để có đủ bằng chứng chẩn đoán, thay vì chỉ kiểm tra trạng thái tức thời sau khi sự cố đã qua.
*Tham khảo lại: Module 19 Phần A, [[../17-Logging-Monitoring/README|Module 17]].*

**Tình huống 6.** Sau khi deploy lại Web01 bằng một bản Docker Compose mới (thay đổi đường dẫn volume mount cho thư mục HTML), truy cập trực tiếp vào Web01 (bỏ qua HAProxy) chỉ thấy trang mặc định của Nginx ("Welcome to nginx!") thay vì nội dung ứng dụng thật.
*Yêu cầu:* xác định vì sao nội dung ứng dụng "biến mất", trong khi container Nginx vẫn khởi động thành công không báo lỗi gì.
*Tham khảo lại: Module 13 (Storage), Module 19 Phần C.*

## Nhóm 3 — Sự cố tầng Database

**Tình huống 7.** Sau một lần chạy lại Ansible playbook để cập nhật cấu hình MySQL (đổi một tham số trong file cấu hình), container MySQL trên VM Database rơi vào trạng thái restart loop liên tục, không bao giờ ở trạng thái `Up` quá 5 giây.
*Yêu cầu:* xác định bước nào trong quy trình cập nhật cấu hình có khả năng cao nhất gây ra restart loop, và cách xác nhận đúng giả thuyết đó bằng bằng chứng cụ thể.
*Tham khảo lại: Module 19 Phần A (kịch bản A2).*

**Tình huống 8.** Team dev báo cáo toàn bộ dữ liệu session người dùng bị mất sau một lần bạn deploy lại Docker Compose stack trên VM Database (chỉ định thay đổi image version của Redis, không có ý định xóa dữ liệu).
*Yêu cầu:* xác định cấu hình nào trong Docker Compose khiến dữ liệu Redis không được giữ lại qua lần deploy, liên hệ với hành vi named volume đã học.
*Tham khảo lại: Module 19 Phần C (kịch bản C4), [[04-Commands]] mục Giai đoạn 4.*

**Tình huống 9.** Bạn phát hiện RabbitMQ Management UI (port 15672) có thể truy cập được từ một máy tính bất kỳ ngoài mạng nội bộ dự án, dù yêu cầu kiến trúc ban đầu ghi rõ port này chỉ nên truy cập nội bộ.
*Yêu cầu:* xác định (những) điểm cấu hình cụ thể nào trong toàn bộ chuỗi Docker Compose + firewall host + cấu hình mạng VM có thể dẫn tới việc port bị lộ ra ngoài ý muốn, và đề xuất cách khắc phục đúng theo nguyên tắc thu hẹp bề mặt tấn công đã học.
*Tham khảo lại: [[../18-Security/README|Module 18]].*

## Nhóm 4 — Sự cố Monitoring

**Tình huống 10.** Dashboard Grafana hiển thị đầy đủ số liệu của Web01, Web02, HAProxy, nhưng riêng phần metrics của VM Database luôn trống, dù bạn xác nhận đã deploy cAdvisor trên node đó giống hệt quy trình đã làm cho 3 node kia.
*Yêu cầu:* xác định vì sao riêng một node không xuất hiện dữ liệu dù quy trình triển khai giống hệt các node còn lại, kiểm tra theo đúng thứ tự đã học ở Module 17/19.
*Tham khảo lại: [[../17-Logging-Monitoring/README|Module 17]], Module 19 Phần B.*

**Tình huống 11.** Trên Prometheus Targets page, target cAdvisor của HAProxy liên tục chuyển trạng thái qua lại giữa `UP` và `DOWN` mỗi vài phút (gọi là "flapping"), trong khi 3 target còn lại luôn ổn định `UP`.
*Yêu cầu:* xác định điều gì đặc biệt ở riêng VM HAProxy (so với 3 node còn lại) có thể gây ra hiện tượng target không ổn định lặp đi lặp lại, thay vì down hẳn hoặc up hẳn.
*Tham khảo lại: [[../17-Logging-Monitoring/README|Module 17]], Module 19 Phần D.*

**Tình huống 12.** Bạn đặt một Alert rule trong Grafana cảnh báo khi RAM một node vượt 85%, nhưng khi Database thực sự chạm 90% RAM trong 10 phút liên tục (do một truy vấn MySQL nặng), không có bất kỳ thông báo nào được gửi tới kênh cảnh báo bạn đã cấu hình.
*Yêu cầu:* liệt kê tối thiểu 3 điểm khác nhau trong chuỗi "điều kiện đúng → alert Firing → gửi thông báo" có thể là nơi xảy ra lỗi, và cách kiểm tra riêng từng điểm.
*Tham khảo lại: [[../17-Logging-Monitoring/README|Module 17]] mục Sự cố 6.*

## Nhóm 5 — Sự cố Backup và Khôi phục

**Tình huống 13.** Bạn kiểm tra lại sau 3 tuần vận hành và phát hiện thư mục backup MySQL trên nơi lưu trữ tập trung không có file mới nào kể từ ngày đầu tiên thiết lập, dù cron job vẫn hiển thị "đã chạy" mỗi ngày trong log hệ thống (không báo lỗi rõ ràng).
*Yêu cầu:* thiết kế cách bạn sẽ điều tra một cron job "chạy nhưng không tạo ra kết quả đúng", phân biệt giữa "job không chạy" và "job chạy nhưng thất bại âm thầm".
*Tham khảo lại: [[04-Commands]] Giai đoạn 6, [[02-Theory]] mục 5.*

**Tình huống 14.** Trong lúc diễn tập khôi phục dữ liệu (disaster recovery drill), bạn khôi phục nhầm một file backup MySQL vào đúng volume của Redis (do đặt tên file backup không đủ rõ ràng để phân biệt). Sau thao tác đó, cả hai service đều không khởi động lại được bình thường.
*Yêu cầu:* xác định cách khôi phục lại đúng trạng thái ban đầu cho cả hai service, và đề xuất một quy ước đặt tên file backup để ngăn sự cố tương tự xảy ra lần nữa.
*Tham khảo lại: [[02-Theory]] mục 5, [[04-Commands]] Giai đoạn 6.*

**Tình huống 15.** RPO bạn đặt ra ban đầu cho dự án là 24 giờ (backup mỗi ngày một lần lúc 2h sáng). Một sự cố mất dữ liệu xảy ra lúc 22h, và khi khôi phục từ bản backup gần nhất (2h sáng cùng ngày), toàn bộ giao dịch từ 2h tới 22h bị mất — đúng như RPO đã cam kết, nhưng đội kinh doanh phản ứng gay gắt vì không biết trước hậu quả cụ thể này.
*Yêu cầu:* đây không phải lỗi kỹ thuật (hệ thống hoạt động đúng như thiết kế) — xác định vấn đề thực sự nằm ở đâu trong quy trình vận hành, và đề xuất cách bạn sẽ tránh lặp lại tình huống tương tự về mặt giao tiếp/quy trình, không chỉ về mặt kỹ thuật.
*Tham khảo lại: [[02-Theory]] mục 5.*

## Nhóm 6 — Sự cố tổng hợp nhiều tầng

**Tình huống 16.** Mỗi ngày vào đúng khung giờ chạy backup (2h sáng), toàn bộ ứng dụng phục vụ traffic (dù rất ít traffic vào giờ đó) đều bị chậm rõ rệt, thậm chí một số request timeout hẳn.
*Yêu cầu:* xác định mối liên hệ giữa tác vụ backup và hiệu năng ứng dụng, đề xuất cách điều chỉnh để backup không ảnh hưởng tới service đang chạy, dù traffic giờ đó thấp.
*Tham khảo lại: [[../19-Troubleshooting/README|Module 19]] Phần D (kịch bản D4).*

**Tình huống 17.** Sau khi bạn áp dụng `--cap-drop=ALL` cho toàn bộ container theo đúng khuyến nghị bảo mật đã học, riêng container RabbitMQ không khởi động được, báo lỗi liên quan tới việc không thể thiết lập một số cấu hình mạng nội bộ của chính nó lúc khởi động.
*Yêu cầu:* xác định capability cụ thể nào RabbitMQ cần lại, và quy trình đúng để xác định điều đó thay vì thử ngẫu nhiên từng capability một.
*Tham khảo lại: [[../18-Security/README|Module 18]], Module 19 Phần A (kịch bản A2).*

**Tình huống 18.** Sau vài tuần vận hành, VM Web01 đột nhiên hết dung lượng đĩa, kéo theo container Nginx không ghi log truy cập được nữa và bắt đầu trả lỗi cho người dùng thật.
*Yêu cầu:* xác định nguyên nhân gốc rễ nhiều khả năng nhất dẫn tới hết dung lượng đĩa trên một node chỉ chạy Nginx tương đối đơn giản, liên hệ lại một bài học cụ thể đã học ở phần Logging.
*Tham khảo lại: [[../17-Logging-Monitoring/README|Module 17]] mục Sự cố 1, Module 19 Phần C.*

**Tình huống 19.** Bạn nhận được cảnh báo Grafana lúc 3h sáng báo Database bị OOM-killed. Khi kiểm tra `docker stats` ngay lúc nhận cảnh báo (đã hồi phục), MySQL container hiển thị RAM usage bình thường, không gần chạm limit đã cấu hình.
*Yêu cầu:* xác định (những) khả năng nào giải thích được hiện tượng "OOM đã xảy ra nhưng số liệu hiện tại lại bình thường", và cách bạn sẽ thu thập đủ bằng chứng để xác nhận đúng khả năng nào đã xảy ra dù sự cố đã qua.
*Tham khảo lại: Module 19 Phần D (kịch bản D1, D3).*

**Tình huống 20.** Bạn được yêu cầu thêm một Web03 vào hệ thống để tăng khả năng chịu tải, mà không được phép gây gián đoạn dịch vụ đang chạy (zero-downtime). Sau khi thêm xong theo đúng quy trình Ansible đã thiết kế ở Bước 4 của [[05-Labs]], HAProxy vẫn không phân phối bất kỳ traffic nào sang Web03 mới, dù `docker ps` trên Web03 xác nhận container đã chạy đúng và healthcheck nội bộ pass bình thường.
*Yêu cầu:* xác định bước nào trong toàn bộ chuỗi "thêm node vào inventory → chạy lại playbook → HAProxy nhận cấu hình mới" có khả năng đã không thực sự xảy ra, kiểm tra bằng chứng cụ thể để xác nhận thay vì chỉ chạy lại playbook nhiều lần và hy vọng nó tự sửa.
*Tham khảo lại: [[04-Commands]] Giai đoạn 4, Module 03 (Playbooks/Templates).*

---

## Cách sử dụng 20 tình huống này hiệu quả nhất

1. Nếu có thể, **tự tạo ra đúng tình huống đó** trên hạ tầng lab thật của bạn (nhiều tình huống có thể tái tạo có chủ đích, ví dụ Tình huống 6, 8, 17) trước khi tự chẩn đoán — trải nghiệm thật luôn dạy nhiều hơn suy luận trên giấy.
2. Với những tình huống khó tái tạo trực tiếp (ví dụ Tình huống 15 mang tính quy trình/giao tiếp hơn kỹ thuật thuần túy), viết ra bằng văn bản cách bạn sẽ tiếp cận, coi như một bài luyện tư duy.
3. Sau khi tự đưa ra kết luận, đối chiếu lại với đúng phần lý thuyết được gợi ý ở "Tham khảo lại" của từng tình huống — nếu kết luận của bạn khác với kiến thức đã học, đây chính là dấu hiệu cần ôn lại module đó kỹ hơn trước khi bước vào công việc thật.
