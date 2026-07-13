---
title: "07 - Interview Questions"
module: 19
tags: [docker, sysops-infra, module-19, troubleshooting, interview]
---

# 07 — Câu hỏi phỏng vấn

## Câu hỏi Junior

1. **Exit code 137 thường có nghĩa là gì?**
   *Đáp án tham khảo:* 128 + 9 = `SIGKILL`, gần như luôn là dấu hiệu container bị OOM Killer của kernel giết do vượt quá giới hạn RAM (của riêng container hoặc của cả host).

2. **Trạng thái `Exited` và `Restarting` khác nhau ở điểm nào?**
   *Đáp án tham khảo:* `Exited` là container đã dừng và giữ nguyên trạng thái đó. `Restarting` là container đang trong vòng lặp tự khởi động lại liên tục theo `restart policy`, thường do crash lặp lại ngay sau mỗi lần khởi động.

3. **Vì sao `curl http://localhost:5432` từ bên trong một container không kết nối được tới container database khác, dù cả hai đều đang chạy?**
   *Đáp án tham khảo:* `localhost` bên trong container luôn trỏ về chính container đó, không bao giờ trỏ sang container khác. Cần dùng đúng tên service (nếu dùng Docker Compose) hoặc tên container làm hostname.

4. **`docker system df` dùng để làm gì?**
   *Đáp án tham khảo:* Xem tổng quan dung lượng đĩa mà Docker đang chiếm dụng, chia theo image, container, volume, và build cache.

5. **Lệnh nào giúp bạn xem cấu hình đầy đủ và trạng thái runtime của một container?**
   *Đáp án tham khảo:* `docker inspect <container>`.

## Câu hỏi Mid-level

6. **Trình bày mô hình debug phân tầng đã học, và giải thích vì sao thứ tự kiểm tra lại quan trọng.**
   *Đáp án tham khảo:* Application → Container → Network → Storage → Host/Kernel. Đi từ ngoài vào trong, loại trừ có bằng chứng ở từng tầng trước khi chuyển sang tầng tiếp theo, tránh lãng phí thời gian kiểm tra sai chỗ hoặc bỏ sót nguyên nhân thật nằm ở tầng khác với nơi triệu chứng biểu hiện.

7. **Named volume trong Docker Compose có bị xóa khi bạn deploy một image mới không? Vì sao nhiều người nhầm đây là bug?**
   *Đáp án tham khảo:* Không. Volume có vòng đời độc lập với container, được thiết kế để giữ nguyên dữ liệu qua các lần deploy/recreate container. Nhầm lẫn thường xảy ra khi người vận hành mong đợi dữ liệu "reset" theo image mới nhưng thực tế dữ liệu database/state vẫn giữ nguyên như thiết kế đúng.

8. **`docker stats` báo CPU trung bình chưa tới 100%, nhưng ứng dụng vẫn bị giật độ trễ liên quan CPU. Bạn sẽ kiểm tra thêm gì?**
   *Đáp án tham khảo:* Đọc trực tiếp `cpu.stat` trong cgroup của container, chú ý trường `nr_throttled`/`throttled_usec` — ứng dụng có thể bị throttle (ngắt quãng) liên tục trong từng chu kỳ ngắn dù số liệu CPU trung bình nhìn qua không cao, do cách CPU quota hoạt động theo chu kỳ thời gian ngắn.

9. **Bind mount một thư mục host vào container chạy bằng non-root user thường gặp lỗi permission gì, và vì sao?**
   *Đáp án tham khảo:* `Permission denied` khi container cố ghi file, do UID bên trong container không trùng UID sở hữu thư mục trên host. Bind mount không tự "dịch" quyền, Linux kiểm tra theo UID số nguyên ở cả hai phía.

## Câu hỏi thực chiến

10. **Bạn nhận được báo cáo "web chậm" lúc 3h chiều, không có thêm thông tin gì khác. Trình bày 5 lệnh đầu tiên bạn chạy, theo đúng thứ tự, và lý do chọn thứ tự đó.**
    *Đáp án tham khảo:* (1) `docker ps -a` — xác nhận mọi container liên quan còn `Up` không, loại trừ khả năng một service đã chết hoàn toàn. (2) `docker stats --no-stream` — xem nhanh có container nào bất thường về tài nguyên không. (3) `docker logs --tail 100 -t` cho các container nghi vấn — tìm lỗi/cảnh báo gần thời điểm báo cáo. (4) Nếu nghi ngờ liên quan hiệu năng, `vmstat 1 5`/`top` đối chiếu tài nguyên tổng thể host. (5) `docker events --since 1h` — kiểm tra có sự kiện bất thường nào (restart, oom, deploy) xảy ra đúng gần thời điểm sự cố không. Thứ tự này ưu tiên bằng chứng rẻ và nhanh trước, tránh đào sâu vào một giả thuyết cụ thể khi chưa có đủ dữ liệu tổng quan.

11. **Bạn phát hiện một container bị OOM-killed, nhưng `--memory` limit của nó vẫn còn dư nhiều so với RAM usage hiển thị trên `docker stats` ngay trước khi bị kill. Bạn giải thích hiện tượng này thế nào, và kiểm tra tiếp ra sao?**
    *Đáp án tham khảo:* Đây là dấu hiệu khả năng cao không phải container này tự vượt limit riêng của nó, mà host tổng thể đã cạn RAM và kernel OOM Killer chọn container này (theo thuật toán tính điểm "đáng bị giết" riêng của kernel, không nhất thiết là tiến trình gây ra vấn đề) làm nạn nhân. Cần kiểm tra `vmstat`/`top`/`free -h` trên host tại đúng thời điểm sự cố, và xem `dmesg` có ghi nhận OOM ở phạm vi toàn hệ thống (system-wide) hay chỉ riêng cgroup của container đó, để xác định đúng đây là vấn đề capacity tổng thể chứ không phải cấu hình sai của riêng container này.

12. **Thiết kế một quy trình debug chuẩn (runbook) ngắn gọn mà một sysadmin trực ca đêm có thể áp dụng ngay khi nhận cảnh báo "container production restart loop", không cần chờ bạn thức dậy hỗ trợ.**
    *Đáp án tham khảo:* Runbook tối thiểu: (1) Chạy `docker events --since 30m` xác nhận đúng là restart loop và tần suất. (2) Chạy `docker logs --tail 200` container đó, tìm dòng lỗi lặp lại ngay trước mỗi lần crash. (3) Đối chiếu thời điểm bắt đầu restart loop với lịch sử deploy gần nhất (Module 16 — image nào vừa được push/pull) — nếu trùng khớp, rollback về image trước đó là hành động an toàn đầu tiên trong đêm, ưu tiên khôi phục dịch vụ trước khi điều tra nguyên nhân sâu. (4) Nếu không liên quan deploy, kiểm tra nhanh dependency (database, service phụ thuộc) có đang down không bằng `docker ps -a` của các service liên quan. (5) Nếu không tự xử lý được trong 15-20 phút theo runbook, escalate theo đúng quy trình on-call thay vì tiếp tục tự mò một mình — runbook tốt luôn có điểm dừng rõ ràng để escalate, không phải cố gắng vô hạn.
