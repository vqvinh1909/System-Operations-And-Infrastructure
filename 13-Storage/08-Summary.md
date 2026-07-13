---
title: "08 - Summary"
module: 13
tags: [docker, sysops-infra, module-13, summary]
---

# 08 - Summary: Storage

## Những gì đã học

- **Vấn đề gốc**: writable layer (OverlayFS) của container không phải nơi lưu dữ liệu lâu dài — mất vĩnh viễn khi `docker rm`.
- **3 loại storage**: Bind Mount (thư mục host tường minh, native speed, kém portable, rủi ro permission cao), Named Volume (Docker quản lý, portable, best practice production), tmpfs (RAM, tốc độ cao nhất, không bền vững, hữu ích cho secret tạm thời).
- **Backup/Restore qua container tạm**: kỹ thuật chuẩn dùng `alpine` + `tar`, mount volume nguồn `:ro`, bind mount thư mục host để lưu kết quả — không cần công cụ ngoài Docker.
- **Anonymous volume là cái bẫy**: `VOLUME` trong Dockerfile không kèm mount tường minh tạo volume tên hash ngẫu nhiên, dễ mất dữ liệu khi `docker rm -v`, dễ tích tụ rác.
- **`-v` vs `--mount`**: `-v` ngắn gọn nhưng có hành vi ngầm định (tự tạo volume mới nếu chưa tồn tại), `--mount` tường minh hơn, khuyến nghị cho script tự động hóa.

## Checklist tự đánh giá

- [ ] Giải thích chính xác vì sao dữ liệu mất khi `docker rm` container không có volume.
- [ ] Chọn đúng loại storage cho từng tình huống cụ thể (dev code, database production, secret tạm thời).
- [ ] Thực hiện được đầy đủ quy trình backup và **restore thành công** một volume — không chỉ backup mà chưa từng test restore.
- [ ] Chẩn đoán và xử lý được lỗi `permission denied` do lệch UID/GID.
- [ ] Nhận diện và tránh bẫy anonymous volume trong mọi cấu hình production.

## Liên kết tới Module tiếp theo

Đến đây, bạn đã nắm đủ 3 trụ cột vận hành container đơn lẻ: Runtime (Module 11), Networking (Module 12), Storage (Module 13). [[../14-Docker-Compose/README|Module 14 - Docker Compose]] sẽ tổng hợp cả ba vào một file cấu hình khai báo duy nhất (`compose.yaml`) — thay vì gõ hàng loạt `docker run`/`docker network create`/`docker volume create` thủ công, bạn định nghĩa toàn bộ stack (services, networks, volumes) một lần, chạy bằng `docker compose up`. Mọi khái niệm named volume, bind mount, user-defined network đã học ở Module 12-13 sẽ xuất hiện lại dưới dạng khai báo YAML, không phải kiến thức mới.

> [!tip] Ghi nhớ cốt lõi
> Container ephemeral là điểm mạnh cho ứng dụng stateless, nhưng là cái bẫy chết người nếu áp dụng nhầm cho dữ liệu cần tồn tại lâu dài. Luôn hỏi trước khi chạy bất kỳ container nào chứa dữ liệu quan trọng: "Nếu tôi `docker rm` container này ngay bây giờ, dữ liệu có còn không?" — nếu câu trả lời không chắc chắn là có, bạn chưa sẵn sàng chạy nó trên production.
