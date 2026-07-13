---
title: "01 - Introduction"
module: 19
tags: [docker, sysops-infra, module-19, troubleshooting]
---

# 01 — Giới thiệu Module: Troubleshooting

## Tình huống mở đầu

3h chiều thứ Sáu. Team dev báo "web bị chậm". Không có thêm thông tin gì khác — không biết chậm ở đâu, từ bao giờ, có phải do deploy gần nhất không. Bạn có 18 module kiến thức trong đầu (Ansible, Docker, Registry, Logging, Security...) nhưng nếu không có một **quy trình** để đi từ "web bị chậm" tới "nguyên nhân cụ thể là gì", kiến thức đó dễ biến thành việc thử ngẫu nhiên từng thứ một — restart container xem có đỡ không, tăng RAM xem có đỡ không — vừa tốn thời gian, vừa không chắc đã sửa đúng gốc rễ.

Module này không dạy thêm công cụ mới. Nó dạy bạn **cách tư duy** khi đối mặt với một sự cố Docker mơ hồ: bắt đầu từ đâu, loại trừ theo thứ tự nào, dùng đúng lệnh nào ở đúng bước nào để không lãng phí thời gian kiểm tra sai chỗ.

## Vì sao module này quan trọng với vai trò Sysadmin/DevOps

Trong 18 module trước, mỗi lần học một công cụ mới (Registry, Logging, Security...) bạn cũng học kèm cách xử lý lỗi **của riêng công cụ đó**. Nhưng sự cố thật trong công việc hiếm khi tự giới thiệu "tôi là lỗi network" hay "tôi là lỗi storage" — nó chỉ biểu hiện ra ngoài bằng một triệu chứng mơ hồ ("chậm", "lỗi 502", "container cứ restart"), và việc đầu tiên bạn phải làm là **khoanh vùng đúng tầng** trước khi áp dụng bất kỳ kỹ năng chuyên sâu nào đã học.

Đây chính là kỹ năng phân biệt rõ nhất một sysadmin/DevOps có kinh nghiệm với người mới: không phải biết nhiều lệnh hơn, mà là biết **thứ tự** dùng lệnh nào để đi thẳng tới nguyên nhân nhanh nhất, dưới áp lực thời gian và thông tin không đầy đủ.

## Mục tiêu học tập

Sau module này, bạn sẽ:

1. Có một mô hình tư duy phân tầng rõ ràng (Application → Container → Network → Storage → Host/Kernel) để khoanh vùng bất kỳ sự cố Docker nào.
2. Thành thạo bộ lệnh chẩn đoán ở từng tầng, biết chính xác lệnh nào trả lời câu hỏi nào.
3. Tự tay gây ra và tự chẩn đoán được các sự cố phổ biến nhất ở cả 4 tầng thông qua lab thực hành.
4. Sẵn sàng cho Module 20 — dự án cuối khóa yêu cầu bạn tự chẩn đoán 20 tình huống sự cố thực tế trên hạ tầng 5 VM, không có đáp án dẫn sẵn từng bước.

## Kiến thức nền cần có trước

- Đã hoàn thành Module 18 (Security).
- Đã học và thực hành đầy đủ Part II (Container Technology, Networking, Storage) và Part III (Registry, Logging & Monitoring, Security) — module này không dạy lại kiến thức nền, mà tổng hợp và luyện tập cách áp dụng chúng để chẩn đoán.
- Đã quen thuộc với `docker inspect`, `docker logs`, `docker stats` ở mức cơ bản từ các module trước.
