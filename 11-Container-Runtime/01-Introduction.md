---
title: "01 - Introduction"
module: 11
tags: [docker, sysops-infra, module-11, introduction]
---

# 01 — Giới thiệu

## Vì sao module này quan trọng hơn bạn nghĩ

Ở Module 10, bạn học cách build một image — tức là đóng gói ứng dụng. Nhưng trong công việc thật của một System Administrator/DevOps Engineer, **build image chỉ chiếm một phần rất nhỏ thời gian**. Phần lớn thời gian bạn dành cho:

- Khởi động và dừng container theo đúng quy trình để không làm mất dữ liệu hay request đang xử lý dở.
- Trả lời câu hỏi "tại sao container này cứ tự restart?" lúc nửa đêm khi có alert.
- Xem log để tìm nguyên nhân lỗi, và biết log driver nào đang được dùng trước khi ổ đĩa server báo đầy.
- Debug một container đang chạy trong production **mà không được phép dừng nó**.
- Giới hạn tài nguyên để một container "tham lam" không làm sập toàn bộ server có 20 container khác đang chạy.
- Dọn dẹp định kỳ hàng chục GB image cũ, container đã exit, build cache tồn đọng — trước khi ổ đĩa hết chỗ.

Đây gọi là **Day-2 Operations** (vận hành ngày thứ hai trở đi) — phân biệt với **Day-1 Operations** (build, deploy lần đầu) của Module 10. Trong ngành, tỷ lệ thời gian dành cho Day-2 so với Day-1 thường lớn hơn nhiều — một hệ thống chạy production trong 2 năm chỉ deploy lần đầu 1 lần, nhưng phải vận hành, giám sát, xử lý sự cố hàng nghìn lần.

## Bối cảnh thực tế: một ca trực điển hình

Hãy tưởng tượng bạn là sysadmin trực ca đêm cho một công ty thương mại điện tử. Đây là những tình huống bạn **sẽ** gặp — không phải "có thể":

1. **2:00 AM** — Hệ thống giám sát báo container `payment-service` restart 15 lần trong 10 phút. Bạn cần biết: đây là crash loop do lỗi code, hay do container bị OOM kill vì vượt giới hạn RAM? Câu trả lời nằm ở `docker inspect` và exit code.
2. **8:00 AM** — Đội vận hành báo server production còn 2% dung lượng ổ đĩa trống. Nguyên nhân: log của một container ghi liên tục không giới hạn suốt 3 tháng, phình lên hàng chục GB. Bạn cần biết cấu hình `max-size`/`max-file` — và đáng lẽ phải cấu hình từ đầu.
3. **11:00 AM** — Developer báo API chậm bất thường. Bạn cần `docker stats` để xem container nào đang ăn CPU/memory cao nhất theo thời gian thực, trước khi kết luận là do code hay do tài nguyên.
4. **3:00 PM** — Cần vào trong một container production đang chạy để kiểm tra file cấu hình, nhưng **không được phép** dừng service hay khởi động lại (đang giờ cao điểm bán hàng). Đây là lúc bạn dùng `docker exec`, không phải `docker attach`.
5. **6:00 PM** — Trước khi hết ca, bạn chạy `docker system df` để biết tổng dung lượng images/containers/volumes/build cache đang chiếm, cân nhắc có nên `prune` không — và phải cân nhắc rất kỹ vì `prune` sai trong production có thể xóa mất image đang được container khác tham chiếu ngầm.

Tất cả 5 tình huống trên đều là nội dung của module này.

## Vị trí module trong bức tranh lớn

```
Module 08 (Linux)  --->  Module 09 (Docker Engine)  --->  Module 10 (Docker Images)
   cgroups, systemd         kiến trúc Docker daemon           build, layer, Dockerfile
        |                                                              |
        |                                                              v
        +------------------------------------------------->  Module 11 (BẠN Ở ĐÂY)
                                                                Container Runtime
                                                          vận hành hằng ngày (Day-2 Ops)
                                                                       |
                                                                       v
                                                              Module 12 (Networking)
                                                          container giao tiếp thế nào
```

Kiến thức cgroups v2 bạn đã học ở Linux Sysadmin Module 08 **không phải lý thuyết suông** — nó chính là cơ chế bên dưới của `--memory`, `--cpus` mà bạn sẽ dùng ở module này. Docker không tự "nghĩ ra" cách giới hạn tài nguyên — nó chỉ là lớp giao diện (CLI) gọi xuống cgroups v2 của kernel Linux.

## Mục tiêu học tập

Sau module này, bạn có thể:

1. Giải thích chính xác từng trạng thái trong vòng đời container và lệnh nào chuyển trạng thái nào.
2. Trình bày cơ chế signal forwarding, giải thích vì sao PID 1 trong container xử lý signal khác tiến trình thường, và vì sao container có thể "treo" 10 giây khi `docker stop` nếu ứng dụng không xử lý `SIGTERM`.
3. Cấu hình giới hạn CPU/memory cho container và dự đoán được điều gì xảy ra khi container vượt giới hạn (bao gồm đọc hiểu exit code 137).
4. Chọn restart policy phù hợp cho từng loại service (stateless web app, database, batch job chạy 1 lần).
5. Cấu hình logging driver để tránh sự cố disk đầy — sự cố production phổ biến bậc nhất liên quan tới Docker.
6. Dùng `docker exec`/`docker inspect --format` thành thạo để debug và trích xuất thông tin cho script tự động hóa.
7. Dùng bộ công cụ giám sát và dọn dẹp: `docker events`, `docker stats`, `docker system df`, `docker system prune` — an toàn, có kiểm soát.

## Cách học module này

Module này **thiên về thực hành**. Lý thuyết ở [[02-Theory]] sẽ giải thích "tại sao" trước, kiến trúc ở [[03-Architecture]] giúp bạn hình dung luồng xử lý, nhưng giá trị thật nằm ở việc gõ lệnh thật trong [[05-Labs]] và đọc kỹ [[06-Troubleshooting]] — đây là những tình huống bạn chắc chắn sẽ gặp lại trong công việc thật, không phải bài tập lý thuyết.

Tiếp theo: [[02-Theory|02 — Lý thuyết]]
