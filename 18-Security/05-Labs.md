---
title: "05 - Labs"
module: 18
tags: [docker, sysops-infra, module-18, security, labs]
---

# 05 — Labs thực hành

## Lab 1: Cài đặt và kiểm chứng Rootless Docker

1. Trên một máy lab (nên dùng VM riêng, không phải máy đang chạy Docker rootful production), cài đặt Rootless Docker theo hướng dẫn ở [[04-Commands]].
2. Chạy `docker info | grep -i rootless` và `ps -o user= -C dockerd` để xác nhận daemon đang chạy bằng user thường.
3. Thử `docker run -d -p 8080:80 nginx:1.27`, xác nhận truy cập được `http://<host>:8080` bình thường.
4. Thử `docker run -d -p 80:80 nginx:1.27` (port < 1024) — quan sát điều gì xảy ra, giải thích vì sao (liên hệ tới việc daemon không còn quyền root để bind port thấp theo cách thông thường).
5. Ghi vào `notes.md`: so sánh cảm nhận về độ trễ mạng (nếu có) giữa rootless và rootful trên cùng máy, dùng thử `curl -w "@curl-format.txt"` hoặc công cụ đo đơn giản.

## Lab 2: Thực hành Capabilities tối thiểu

1. Chạy một ứng dụng thật (ví dụ `nginx:1.27`) với `--cap-drop=ALL`, xác nhận ứng dụng vẫn hoạt động bình thường qua port mapping của Docker.
2. Chạy thử một container cố tình cần `NET_ADMIN` (ví dụ container thực hiện thao tác `ip link` bên trong) mà **không** cấp capability đó — quan sát lỗi `Operation not permitted`.
3. Cấp lại đúng `--cap-add=NET_ADMIN`, xác nhận thao tác chạy được.
4. Ghi vào `notes.md` bảng đối chiếu: ứng dụng nào trong các container bạn từng chạy ở Part II/III cần capability gì thật sự, dựa trên việc bạn tự test bằng cách drop hết rồi add dần.

## Lab 3: Seccomp và AppArmor

1. Chạy một container với `--security-opt seccomp=unconfined` và thử gọi một syscall bị chặn bởi seccomp mặc định (ví dụ dùng `strace` hoặc một script Python gọi `ptrace`) để thấy nó **chạy được**.
2. Bỏ cờ `unconfined`, chạy lại với seccomp mặc định của Docker, thử lại đúng thao tác đó — quan sát lỗi bị chặn.
3. Kiểm tra AppArmor profile đang gán cho một container đang chạy (`docker inspect --format '{{.AppArmorProfile}}' <container>`).
4. Thử chạy container với `--security-opt apparmor=unconfined`, thử ghi file vào một đường dẫn thường bị AppArmor `docker-default` chặn, so sánh với khi chạy có AppArmor mặc định.

## Lab 4: Image Scanning và Docker Bench for Security

1. Chọn một image cũ hơn 1 năm (ví dụ một base image version cũ có sẵn nhiều CVE đã biết), scan bằng cả `docker scout cves` và `trivy image`, so sánh kết quả hai công cụ.
2. Viết một Dockerfile tối thiểu, build image, scan lại — xác nhận số lượng CVE giảm đáng kể khi dùng base image mới hơn/tối giản hơn (ví dụ chuyển từ `ubuntu:latest` sang `alpine` hoặc distroless nếu phù hợp ứng dụng).
3. Clone và chạy Docker Bench for Security theo đúng hướng dẫn ở [[04-Commands]] trên máy lab bạn đang dùng cho toàn khóa học.
4. Chọn ra 5 mục `[WARN]` bạn cho là quan trọng nhất, tự khắc phục từng mục, chạy lại Docker Bench để xác nhận chuyển sang `[PASS]`. Ghi lại vào `notes.md` từng mục: nội dung cảnh báo, cách bạn khắc phục, và vì sao mục đó quan trọng với môi trường thật.

## Tiêu chí hoàn thành

- Có một máy lab chạy Rootless Docker hoạt động ổn định, xác nhận được bằng lệnh kiểm tra UID của daemon.
- Viết được cấu hình `--cap-drop=ALL` kèm `--cap-add` tối thiểu đúng cho ít nhất 2 ứng dụng khác nhau, có giải thích lý do từng capability được thêm lại.
- Giải thích lại được bằng lời của mình sự khác biệt thực tế (không chỉ lý thuyết) giữa seccomp bị chặn và AppArmor bị chặn, dựa trên đúng những gì quan sát được trong Lab 3.
- Có kết quả chạy Docker Bench for Security trước/sau khi khắc phục ít nhất 5 mục cảnh báo, lưu lại làm bằng chứng trong `labs/`.
