---
title: "07 - Interview Questions"
module: 16
tags: [docker, sysops-infra, module-16, registry, harbor, interview]
---

# 07 — Câu hỏi phỏng vấn

## Câu hỏi Junior

1. **Docker Hub và Private Registry khác nhau ở điểm cơ bản nào?**
   *Đáp án tham khảo:* Docker Hub là registry công khai do Docker Inc. vận hành, mặc định mọi client Docker trỏ tới. Private Registry là registry do chính doanh nghiệp tự host trong mạng nội bộ, kiểm soát toàn quyền truy cập và dữ liệu.

2. **`registry:2` là gì?**
   *Đáp án tham khảo:* Image chính thức của Docker chạy dự án mã nguồn mở Distribution — một registry tuân thủ chuẩn OCI Distribution Spec, cung cấp API push/pull cơ bản, không có UI hay RBAC tích hợp.

3. **Vì sao khi push lên registry nội bộ phải gắn tag dạng `registry.company.local/app:v1` thay vì chỉ `app:v1`?**
   *Đáp án tham khảo:* Docker mặc định hiểu tag không có domain là thuộc Docker Hub. Phải khai báo rõ domain/host của registry đích trong tag để Docker biết push/pull đi đâu.

4. **Harbor là gì?**
   *Đáp án tham khảo:* Một registry doanh nghiệp mã nguồn mở, dự án CNCF Graduated, bọc thêm lớp quản trị (UI, RBAC theo project, image scanning tích hợp, replication) lên trên engine Distribution.

5. **Image scanning dùng để làm gì?**
   *Đáp án tham khảo:* Phân tích các package/thư viện trong image, đối chiếu với vulnerability database (CVE đã biết) để phát hiện lỗ hổng bảo mật trước khi image được deploy.

## Câu hỏi Mid-level

6. **Giải thích vì sao doanh nghiệp không nên dùng thẳng Docker Hub public để lưu image nội bộ, kể cả khi trả tiền cho private repo.**
   *Đáp án tham khảo:* Bốn lý do chính: rủi ro rò rỉ dữ liệu nhạy cảm nằm trong layer, thiếu RBAC chi tiết theo team/project, thiếu audit log phục vụ compliance, và chi phí/độ trễ băng thông khi nhiều server pull qua internet thay vì cùng mạng nội bộ.

7. **Xóa một tag trên Harbor có giải phóng dung lượng đĩa ngay lập tức không? Giải thích.**
   *Đáp án tham khảo:* Không. Xóa tag chỉ đánh dấu manifest là unreferenced, blob thật vẫn còn trên Storage Backend. Dung lượng chỉ giải phóng sau khi chạy Garbage Collection, quét và xóa các blob không còn manifest nào tham chiếu.

8. **Robot account trong Harbor khác gì tài khoản người dùng thông thường, và vì sao nên dùng nó cho CI/CD?**
   *Đáp án tham khảo:* Robot account là tài khoản kỹ thuật, được cấp quyền giới hạn theo project cụ thể, không gắn với một người dùng thật. Dùng cho CI/CD để tránh nhúng mật khẩu cá nhân vào pipeline, dễ thu hồi quyền độc lập mà không ảnh hưởng tài khoản người dùng thật.

9. **Vì sao một image "sạch" lúc build có thể bị đánh dấu "có lỗ hổng" vài tuần sau dù không hề thay đổi?**
   *Đáp án tham khảo:* Vulnerability database được cập nhật liên tục — CVE mới được phát hiện cho các package đã tồn tại sẵn trong image từ trước. Đây là lý do cần re-scan định kỳ, không chỉ scan một lần lúc push.

## Câu hỏi thực chiến

10. **Bạn `docker push` vào Harbor và nhận lỗi `unauthorized: authentication required`, dù chắc chắn đã `docker login` thành công vài phút trước. Bạn chẩn đoán và xử lý thế nào?**
    *Đáp án tham khảo:* Kiểm tra lại đúng registry/project trong tag image (có thể đang push nhầm project không có quyền). Kiểm tra vai trò user trong project trên Harbor Portal — nếu chỉ là Guest (read-only) thì không thể push dù login thành công. Nếu vẫn không rõ, login lại để loại trừ khả năng session hết hạn.

11. **Container `harbor-core` liên tục restart sau khi bạn sửa `harbor.yml` và chạy lại `docker compose up`. Vì sao, và quy trình đúng để áp dụng thay đổi cấu hình là gì?**
    *Đáp án tham khảo:* `harbor.yml` không được các container đọc trực tiếp lúc runtime — nó chỉ là input cho script `prepare` sinh ra file cấu hình thật cho từng container. Sửa `harbor.yml` rồi chạy thẳng `docker compose up` mà bỏ qua bước `./prepare` sẽ khiến các container dùng cấu hình cũ hoặc không nhất quán, gây lỗi khởi động. Quy trình đúng: sửa `harbor.yml` → chạy `./prepare` → `docker compose up -d`.

12. **Bạn cần thiết kế chính sách bảo mật image cho một pipeline CI/CD thực tế: build → push → deploy. Bạn sẽ đặt bước image scanning ở đâu trong pipeline, và policy nên xử lý thế nào khi phát hiện lỗ hổng mức Critical?**
    *Đáp án tham khảo:* Đặt scan ngay sau bước push, trước bước deploy — biến scan thành một gate bắt buộc (không chỉ chạy nền tham khảo). Với lỗ hổng mức Critical, cấu hình policy "Prevent vulnerable images from running" để Harbor tự động chặn pull/deploy, đồng thời pipeline CI/CD nên fail build và thông báo cho team — không để việc phát hiện Critical CVE chỉ dừng ở một dòng log không ai đọc.

13. **Registry nội bộ của bạn báo `no space left on device`, nhưng team dev đang cần deploy gấp một hotfix production. Bạn xử lý theo thứ tự ưu tiên nào?**
    *Đáp án tham khảo:* Trước tiên kiểm tra nhanh dung lượng còn trống thật sự (`df -h`) để xác nhận đúng nguyên nhân. Nếu đúng do registry đầy, ưu tiên xóa các tag rõ ràng không còn giá trị (build test cũ, nhánh đã merge) rồi chạy Garbage Collection ngay để giải phóng dung lượng đủ cho hotfix đi qua trước — xử lý retention policy dài hạn (tự động dọn định kỳ) là việc làm sau khi sự cố khẩn cấp đã qua, tránh vừa dọn vội vừa risk xóa nhầm image đang cần.
