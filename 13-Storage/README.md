---
title: "Module 13 - Storage"
tags: [docker, sysops-infra, module-13, storage, volume]
module: 13
part: "II - Container Fundamentals"
difficulty: Intermediate
status: draft
created: 2026-07-12
prerequisites: ["[[../12-Networking/README|Module 12]]"]
next: "[[../14-Docker-Compose/README|Module 14]]"
---

# Module 13 - Storage

## Vì sao module này quan trọng

Container vốn sinh ra để **ephemeral** — dùng xong xóa, không tiếc. Nhưng dữ liệu bên trong (database, file upload, log, session) thì không thể "xóa xong quên luôn". Đây chính là mâu thuẫn cốt lõi mà bất kỳ sysadmin nào vận hành Docker trong môi trường thật đều phải giải quyết: **làm sao để dữ liệu sống lâu hơn vòng đời của container?**

Module 12 đã cho Vinh thấy container giao tiếp với nhau và với bên ngoài qua network như thế nào. Module này giải quyết vế còn lại: container **lưu trữ và bảo vệ dữ liệu** như thế nào. Đây là chủ đề mà một sysadmin non kinh nghiệm dễ mắc lỗi nhất — và hậu quả thường là **mất dữ liệu production**, thứ không có nút "undo".

## Nội dung module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Giới thiệu, mục tiêu học tập, tại sao storage là chủ đề rủi ro cao |
| [[02-Theory]] | Lý thuyết chi tiết: bind mount, volume, tmpfs, Internal Working (mount namespace, OverlayFS) |
| [[03-Architecture]] | Sơ đồ ASCII so sánh 3 loại storage và luồng backup/restore |
| [[04-Commands]] | Toàn bộ lệnh: `docker volume`, cú pháp `-v` / `--mount`, backup/restore |
| [[05-Labs]] | Lab thực hành từ cơ bản đến mini project backup/restore database |
| [[06-Troubleshooting]] | Các lỗi thực tế: permission denied, mất data, disk đầy |
| [[07-Interview]] | Câu hỏi phỏng vấn: Junior / Mid-level / Thực chiến |
| [[08-Summary]] | Tóm tắt và cầu nối sang Module 14 - Docker Compose |
| [[labs/README|labs/]] | Nơi lưu code, script, config dùng trong lab |
| [[scripts/README|scripts/]] | Nơi lưu script tự viết (backup, restore, cleanup) |

## Mục tiêu sau khi hoàn thành module

Sau module này, Vinh phải:

1. Phân biệt rạch ròi 3 loại storage của Docker (bind mount, named volume, tmpfs) và biết **khi nào dùng loại nào** trong tình huống thực tế.
2. Giải thích được vì sao named volume là best practice cho dữ liệu production (database), không phải bind mount.
3. Thực hiện được backup và restore volume bằng container tạm — đây là kỹ năng **bắt buộc phải thuộc lòng** vì nó xuất hiện trong hầu hết quy trình vận hành Docker thật.
4. Nhận diện và xử lý được các lỗi storage thường gặp: permission denied do lệch UID/GID, mất data do nhầm anonymous volume, disk đầy do volume rác không dọn.
5. Hiểu Internal Working: named volume và bind mount đi vào container qua cơ chế nào bên dưới OverlayFS (đã học ở Module 08).

## Điều kiện tiên quyết

- Đã hoàn thành [[../12-Networking/README|Module 12 - Networking]] (hiểu container network cơ bản).
- Đã học Linux Sysadmin: permission, UID/GID, filesystem, mount namespace.
- Đã cài Docker Engine (khuyến nghị bản 29.x hiện hành, ví dụ 29.6.1) để thực hành lab.

---

> [!note] Self-Review
> - Đã WebSearch xác nhận: Docker Engine nhánh 29.x hiện hành (29.6.1, phát hành 6/2026); named volume mặc định lưu tại `/var/lib/docker/volumes/<tên>/_data` trên Linux host. Không có số liệu nào trong module bị bịa — mọi con số phiên bản đều theo facts sheet đã xác nhận.
> - Đã đo 2 sơ đồ ASCII trong [[03-Architecture]] bằng script Python đo độ dài từng dòng khung (`+`/`|`) theo đúng quy trình bắt buộc — cả hai đều cho kết quả **OK**, không có khung nào lệch padding.
