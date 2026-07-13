---
title: "01 - Introduction"
module: 8
tags: [docker, sysops-infra, module-08, introduction]
---

# 01 - Introduction: Vì sao Container ra đời

## Bài toán doanh nghiệp thật

Hồi 2010-2013, một công ty fintech điển hình vận hành thế này: team dev code trên máy Windows/Mac với PHP 5.6 và MySQL 5.5 cài local; khi deploy lên server production (CentOS 6, PHP 5.4, MySQL 5.1) thì lỗi tùm lum vì phiên bản thư viện lệch nhau. Câu cửa miệng nổi tiếng của dev lúc đó: **"It works on my machine"** — và câu trả lời cay đắng của ops: **"Vậy thì ship luôn cái machine của mày đi."**

Giải pháp đầu tiên ngành dùng là **Virtual Machine**: đóng gói cả OS + app thành 1 file `.ova`/`.vmdk`, ship nguyên VM đó cho môi trường khác. Nó giải quyết được vấn đề "works on my machine", nhưng đẻ ra vấn đề mới:

- 1 VM cõng nguyên 1 Guest OS (kernel riêng, `systemd` riêng, dễ tốn 1-2 GB RAM chỉ để OS chạy không) → chạy 20 microservice = 20 VM = tốn tài nguyên khủng khiếp.
- Boot 1 VM mất 30-60 giây (phải qua BIOS ảo, bootloader, init hệ thống) → không phù hợp scale nhanh theo traffic.
- Image VM nặng hàng GB, kéo qua mạng chậm, versioning khó.

## Container giải quyết đúng điểm đau nào

Container ra đời (khái niệm cô lập tiến trình đã có từ `chroot` 1979, FreeBSD Jails 2000, Solaris Zones 2004, LXC 2008; Docker năm 2013 mới đóng gói lại thành sản phẩm dễ dùng) để trả lời câu hỏi: **"Làm sao cô lập ứng dụng như VM, nhưng không phải cõng nguyên một OS mới mỗi lần?"**

Câu trả lời: **container không ảo hóa phần cứng, nó ảo hóa ở mức hệ điều hành (OS-level virtualization)**. Mọi container trên một máy host đều **dùng chung 1 kernel Linux của host**, chỉ khác nhau ở "cái nhìn" (view) về tiến trình, mạng, filesystem — cái nhìn đó do namespace tạo ra, còn giới hạn tài nguyên do cgroup quản lý.

Hệ quả trực tiếp:

| Tiêu chí | Virtual Machine | Container |
|---|---|---|
| Thời gian khởi động | 30-60 giây | Dưới 1 giây (thực chất chỉ là `fork` + `exec` tiến trình) |
| Kích thước image | Hàng GB (cả OS) | Vài MB - vài trăm MB (chỉ app + dependency) |
| Mật độ trên 1 host | Chục VM | Hàng trăm container |
| Cách ly | Mạnh (kernel riêng) | Yếu hơn (chung kernel host) |
| Portable | Cần hypervisor tương thích | Chạy được mọi nơi có container runtime |

## Ví dụ doanh nghiệp cụ thể

Một sàn thương mại điện tử vào mùa sale 11/11 cần scale service "giỏ hàng" từ 5 lên 200 instance trong vòng 2 phút khi traffic tăng đột biến. Nếu dùng VM, autoscaling group phải boot 195 VM mới — quá chậm, khách hủy giỏ hàng trước khi hệ thống kịp phản hồi. Dùng container, việc scale chỉ là tạo thêm 195 tiến trình mới từ cùng 1 image đã cache sẵn trên node — vài giây là xong. Đây là lý do container trở thành nền tảng bắt buộc cho kiến trúc microservice và cloud-native, và là bước đệm trực tiếp trước khi học Kubernetes ở các khóa sau.

## Container KHÔNG phải viên đạn bạc

Sysadmin có kinh nghiệm cần tỉnh táo: container **không cách ly mạnh bằng VM** vì chung kernel — một lỗ hổng kernel (privilege escalation) có thể cho phép tiến trình trong container thoát ra host (container escape). Vì vậy các workload cần cách ly bảo mật cực cao (multi-tenant chạy code không tin cậy) vẫn dùng VM, hoặc kết hợp container-trong-VM (gVisor, Kata Containers, Firecracker microVM). Đây là lý do câu trả lời phỏng vấn "container an toàn hơn VM" là **sai** — chính xác phải là "container gọn nhẹ hơn, nhưng cách ly yếu hơn VM".

## Vị trí module trong khóa học

Đây là module đầu tiên của **Part II - Container Fundamentals**. Module 07 (Enterprise Automation) đã khép lại Part I về Ansible. Từ module này, ta chuyển sang container: Module 08 dạy "tại sao nó hoạt động như vậy" (kernel internals), Module 09 trở đi dạy "làm thế nào" (Docker CLI, image, compose). Bỏ qua module này, bạn vẫn gõ được lệnh Docker, nhưng sẽ bó tay khi container lỗi lạ — container không resolve DNS, container ăn hết RAM host, permission bên trong container không khớp với host.

## Điều kiện tiên quyết

- Đã hoàn thành [[../07-Enterprise-Automation/README|Module 07]]: hiểu process, PID, `/proc`, filesystem mount, permission Linux cơ bản.
- Có quyền `sudo` trên một máy Linux (VM Ubuntu 24.04 hoặc tương đương) để chạy lệnh demo trong module — khuyến nghị VM riêng, không thử nghiệm namespace trên server production.
- Chưa cần cài Docker ở module này — demo namespace/cgroup dùng công cụ có sẵn trong `util-linux` (`unshare`, `nsenter`).

## Cách học module hiệu quả

1. Đọc [[02-Theory]] để hiểu khái niệm và cơ chế bên trong.
2. Xem [[03-Architecture]] để hình dung sơ đồ so sánh VM vs Container, cấu trúc namespace, OverlayFS.
3. Chạy tay từng lệnh trong [[04-Commands]].
4. Làm [[05-Labs]] — phần cuối là tự dựng "mini container" bằng `unshare` + `chroot`, trải nghiệm đúng những gì Docker làm bên dưới.
5. Đọc [[06-Troubleshooting]] để biết lỗi thực tế liên quan namespace/cgroup khi vận hành Docker sau này.
6. Ôn [[07-Interview]] trước khi phỏng vấn vị trí có Docker/container trong JD.

Sang [[08-Summary]] để biết module này kết nối thế nào với [[../09-Docker-Engine/README|Module 09]].
