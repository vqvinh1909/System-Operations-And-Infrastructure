---
title: "01 - Introduction"
module: 13
tags: [docker, sysops-infra, module-13, introduction]
---

# 01 - Introduction: Storage trong Docker

## Tại sao storage lại là chủ đề "rủi ro cao" nhất trong Docker

Hãy hình dung tình huống này — nó không hiếm trong môi trường doanh nghiệp thật:

Một sysadmin junior được giao restart lại container database vì nó bị treo. Anh ta chạy `docker rm -f mydb` rồi `docker run` lại y hệt lệnh cũ. Container chạy lên bình thường, healthcheck xanh, ứng dụng kết nối được... nhưng **toàn bộ dữ liệu khách hàng biến mất**. Vì sao? Vì lệnh `docker run` gốc không có `-v`, dữ liệu nằm trong **writable layer** của container (dùng OverlayFS, Vinh đã học ở Module 08) — khi container bị xóa, layer đó bị xóa theo, không có cách nào phục hồi.

Đây chính là lý do storage là chủ đề mà một sysadmin vận hành Docker **không được phép hiểu mơ hồ**. Container được thiết kế để **ephemeral** (dùng xong bỏ) — đó là ưu điểm cho việc scale, rolling update, immutable infrastructure. Nhưng ưu điểm đó trở thành thảm họa nếu áp dụng nhầm cho dữ liệu cần tồn tại lâu dài (database, file người dùng upload, log audit).

## Vấn đề cốt lõi: Container filesystem không được thiết kế để lưu dữ liệu lâu dài

Nhắc lại từ Module 08: mỗi container có một **writable layer** nằm trên cùng chồng các layer chỉ đọc (read-only) của image, ghép lại bằng OverlayFS. Writable layer này:

- Chỉ tồn tại trong vòng đời của container đó.
- Bị xóa vĩnh viễn khi chạy `docker rm`.
- Hiệu năng I/O kém hơn so với truy cập trực tiếp filesystem host (do phải đi qua cơ chế copy-on-write của OverlayFS).

Vì vậy Docker cung cấp 3 cơ chế để "thoát" dữ liệu ra khỏi writable layer, mỗi cơ chế phù hợp một use case khác nhau:

1. **Bind Mount** — mượn trực tiếp một thư mục/file có sẵn trên host.
2. **Named Volume** — nhờ Docker tạo và quản lý một vùng lưu trữ riêng.
3. **tmpfs Mount** — lưu vào RAM, không bao giờ chạm đĩa.

Module này sẽ đi sâu vào cả ba, giải thích tại sao chọn cái này thay vì cái kia trong từng tình huống thực tế, và quan trọng nhất: **cách backup/restore volume** — kỹ năng sống còn khi vận hành container có state (stateful) trong production.

## Mục tiêu học tập

Sau khi hoàn thành module này, Vinh có thể:

- [ ] Giải thích sự khác biệt về cơ chế, hiệu năng, và vòng đời giữa bind mount, named volume, tmpfs.
- [ ] Chọn đúng loại storage cho từng tình huống: code dev, database production, secret tạm thời.
- [ ] Dùng thành thạo `docker volume create/ls/inspect/rm` và cú pháp `-v` lẫn `--mount`.
- [ ] Backup một volume ra file `.tar.gz` và restore lại vào volume mới bằng container tạm — không cần cài thêm công cụ nào ngoài Docker.
- [ ] Debug được lỗi `permission denied` khi bind mount do lệch UID/GID giữa host và container.
- [ ] Nhận diện và tránh bẫy "anonymous volume" gây mất dữ liệu tưởng chừng vô lý.
- [ ] Hiểu được cách named volume "né" OverlayFS write layer để đảm bảo hiệu năng và persistence.

## Vì sao học xong module này thì học Ansible sẽ dễ hơn

Vinh đang học song song Ansible. Một trong những playbook phổ biến nhất trong thực tế doanh nghiệp là tự động hóa việc deploy container có kèm volume (ví dụ deploy một stack database + backup cron). Nắm chắc khái niệm volume, bind mount, cú pháp `-v`/`--mount` ở module này sẽ giúp Vinh viết Ansible task với module `community.docker.docker_container` chính xác hơn nhiều, tránh việc viết playbook "chạy được" nhưng âm thầm mất dữ liệu do cấu hình volume sai.

## Liên kết

- Trước: [[../12-Networking/README|Module 12 - Networking]]
- Tiếp theo trong module này: [[02-Theory]]
