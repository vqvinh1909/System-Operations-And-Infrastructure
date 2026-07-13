---
title: "07 - Interview Questions"
module: 18
tags: [docker, sysops-infra, module-18, security, interview]
---

# 07 — Câu hỏi phỏng vấn

## Câu hỏi Junior

1. **Vì sao Docker mặc định (rootful) được coi là một rủi ro bảo mật kiến trúc?**
   *Đáp án tham khảo:* Docker daemon mặc định chạy bằng root (UID 0) trên host. Mọi container là tiến trình con của daemon đó — nếu kẻ tấn công thoát được khỏi namespace cô lập của container (container breakout), họ có thể chạm tới quyền root thật trên host.

2. **Rootless Docker giải quyết vấn đề trên bằng cách nào?**
   *Đáp án tham khảo:* Chạy toàn bộ daemon Docker bằng một user thường (không phải root), dùng user namespace để ánh xạ UID/GID, kết hợp RootlessKit và slirp4netns cho phần networking không cần quyền root.

3. **`--cap-drop=ALL` dùng để làm gì?**
   *Đáp án tham khảo:* Loại bỏ toàn bộ Linux capability (các mảnh quyền root nhỏ) mà Docker mặc định cấp cho container, áp dụng nguyên tắc least privilege — chỉ cấp lại đúng capability nào ứng dụng thật sự cần bằng `--cap-add`.

4. **Seccomp chặn ở tầng nào?**
   *Đáp án tham khảo:* Tầng system call (syscall) — giới hạn tập lệnh gọi tới kernel mà tiến trình trong container được phép thực hiện.

5. **Image scanning dùng để phát hiện gì?**
   *Đáp án tham khảo:* Các lỗ hổng bảo mật đã biết (CVE) trong package/thư viện nằm bên trong image, bằng cách đối chiếu với vulnerability database.

## Câu hỏi Mid-level

6. **Seccomp và AppArmor khác nhau ở điểm nào, vì sao cần dùng cả hai thay vì chỉ một?**
   *Đáp án tham khảo:* Seccomp chặn ở mức "có được gọi hành động (syscall) này không", AppArmor chặn ở mức "có được chạm vào tài nguyên (file, network, capability) cụ thể này không". Hai cơ chế bổ sung, không trùng lặp — dùng cả hai tạo ra nhiều lớp phòng thủ độc lập, một lỗ hổng vượt qua được lớp này chưa chắc vượt qua được lớp kia.

7. **Vì sao tập capability mặc định của Docker (khoảng 14 capability) vẫn được coi là chưa đủ an toàn với nhiều ứng dụng?**
   *Đáp án tham khảo:* Tập mặc định được thiết kế đủ rộng để phần lớn ứng dụng chạy được ngay mà không cần cấu hình thêm, nhưng phần lớn ứng dụng web/API thông thường thực tế không cần bất kỳ capability nào trong số đó — nguyên tắc least privilege yêu cầu thu hẹp tối đa, không dừng lại ở "mặc định của nhà cung cấp".

8. **Trình bày quy trình bạn sẽ đưa image scanning trở thành một gate thật sự trong pipeline CI/CD, không chỉ chạy tham khảo.**
   *Đáp án tham khảo:* Đặt bước scan ngay sau build, trước push/deploy. Cấu hình công cụ scan trả về exit code khác 0 khi phát hiện CVE vượt ngưỡng nghiêm trọng đã định nghĩa (ví dụ Critical/High chưa có fix), khiến CI/CD job tự động fail tại bước đó — ngăn không cho image lỗi đi tiếp, thay vì chỉ ghi log cảnh báo.

9. **Docker Bench for Security kiểm tra dựa theo chuẩn nào, và kết quả `[WARN]` có luôn nghĩa là phải sửa ngay không?**
   *Đáp án tham khảo:* Dựa theo CIS Docker Benchmark. `[WARN]` không đồng nghĩa bắt buộc sửa ngay — nhiều mục mang tính khuyến nghị theo ngữ cảnh (ví dụ khuyến nghị liên quan Swarm không áp dụng nếu không dùng Swarm), cần đọc kỹ và đối chiếu với hạ tầng thật trước khi quyết định.

## Câu hỏi thực chiến

10. **Một ứng dụng production sau khi bạn thêm `--cap-drop=ALL` bị lỗi ngay lúc khởi động với thông báo không rõ ràng. Bạn chẩn đoán và xử lý theo quy trình nào để không phải revert hoàn toàn về mặc định?**
    *Đáp án tham khảo:* Chạy lại container ở chế độ tương tác (`docker run --rm -it --cap-drop=ALL <image> sh`) để quan sát trực tiếp bước nào lỗi. Đọc kỹ thông báo lỗi tìm dấu hiệu liên quan tới hành động cần quyền đặc biệt (chown, bind port thấp, đổi UID...). Xác định đúng capability tương ứng, thêm lại bằng `--cap-add` chỉ đúng capability đó, giữ nguyên phần còn lại đã drop — không mở lại toàn bộ.

11. **Bạn phát hiện một image production đang chạy có CVE mức Critical đã tồn tại nhiều tháng, nhưng bản vá (fixed version) của package đó phá vỡ tương thích với phần code hiện tại. Bạn xử lý tình huống này như thế nào?**
    *Đáp án tham khảo:* Không thể chỉ "chờ dev sửa code" mà bỏ mặc rủi ro Critical đang chạy production. Đánh giá các lớp phòng thủ bù đắp tạm thời (compensating controls): thu hẹp capability/seccomp/AppArmor của container đó chặt hơn nữa, giới hạn network exposure (không public trực tiếp ra internet nếu có thể), tăng giám sát log/metric riêng cho container này để phát hiện sớm nếu bị khai thác. Đồng thời báo cáo rõ ràng cho team liên quan về rủi ro đang chấp nhận tạm thời và timeline dự kiến để sửa gốc rễ — không để rủi ro Critical tồn tại âm thầm không ai theo dõi.

12. **Một đồng nghiệp đề xuất tắt hẳn AppArmor và seccomp trên toàn bộ host để "giảm phiền phức khi debug lỗi container liên tục". Bạn phản hồi thế nào?**
    *Đáp án tham khảo:* Không đồng ý làm vậy trên môi trường production/gần production — tắt cả hai lớp phòng thủ độc lập cùng lúc chỉ để tiện debug là đánh đổi bảo mật lấy sự tiện lợi tạm thời, trong khi vấn đề gốc rễ (lỗi cấu hình cụ thể) hoàn toàn có thể chẩn đoán bằng cách đọc log AppArmor/seccomp (`dmesg`, `journalctl -k`) đã học ở [[06-Troubleshooting]], thay vì tắt mù toàn bộ cơ chế. Nếu cần tắt tạm thời để cô lập nguyên nhân lỗi, chỉ nên làm trên môi trường lab/dev riêng biệt, không bao giờ áp dụng thay đổi đó lên production.
