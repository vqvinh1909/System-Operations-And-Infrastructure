---
title: "05 - Labs"
module: 19
tags: [docker, sysops-infra, module-19, troubleshooting, labs]
---

# 05 — Labs thực hành

> [!info] Cách làm lab của module này
> Khác với các module trước, mỗi lab dưới đây yêu cầu bạn **tự gây ra lỗi có kiểm soát** trước, sau đó tự chẩn đoán bằng đúng quy trình đã học ở [[02-Theory]] và bộ lệnh ở [[04-Commands]] — không đọc trước đáp án trong [[06-Troubleshooting]]. Đây là cách luyện tập gần nhất với tình huống thật, nơi không ai đưa sẵn cho bạn nguyên nhân.

## Lab 1: Container Debugging

1. Tạo một image cố tình lỗi entrypoint (ví dụ Dockerfile với `CMD ["nonexistent-binary"]`), build và chạy — quan sát trạng thái, tự tìm exit code và nguyên nhân bằng `docker inspect`/`docker logs`.
2. Tạo một container cố tình restart loop (ví dụ entrypoint là script `exit 1` ngay lập tức, với `restart: always`). Dùng `docker events` để đếm số lần restart trong 1 phút, đối chiếu với `RestartCount`.
3. Thêm `HEALTHCHECK` vào một image, cố tình làm healthcheck fail (ví dụ kiểm tra một endpoint không tồn tại), quan sát trạng thái chuyển từ `starting` → `unhealthy`.

## Lab 2: Network Debugging

1. Tạo 2 container trên **2 network khác nhau** (không dùng `--network` chung), thử ping giữa chúng — xác nhận lỗi, tự chẩn đoán bằng `docker network inspect` theo đúng quy trình 4 bước ở [[03-Architecture]].
2. Sửa lại để cả hai cùng network, ping lại thành công — ghi chú lại sự khác biệt bạn quan sát được ở từng bước.
3. Cố tình publish 2 container cùng lên một port host (`-p 8080:80` cho cả hai) — quan sát lỗi Docker báo khi chạy container thứ hai, giải thích nguyên nhân.

## Lab 3: Storage Debugging

1. Chạy một container với image chạy bằng non-root user (ví dụ user UID 1000), bind mount một thư mục host thuộc sở hữu `root`. Quan sát lỗi `Permission denied` khi ứng dụng cố ghi file, tự chẩn đoán bằng cách so sánh UID giữa `docker exec <container> id` và `ls -lan` trên host.
2. Sửa quyền thư mục host (`chown`) cho khớp UID container, xác nhận ghi file thành công.
3. Cố tình làm đầy dung lượng ổ đĩa lab (dùng `fallocate` hoặc container ghi file lớn liên tục vào volume), quan sát Docker báo lỗi gì khi cố `docker run` container mới lúc đĩa gần đầy.

## Lab 4: Performance Analysis

1. Chạy một container với `--memory=50m` chạy ứng dụng cố tình cấp phát nhiều RAM hơn giới hạn đó (ví dụ script Python liên tục append vào một list lớn). Quan sát container bị kill, xác nhận bằng `docker inspect --format '{{.State.OOMKilled}}'` và `dmesg -T | grep -i killed`.
2. Chạy một container `--cpus=0.5` cùng lúc với một tải công việc nặng CPU (`stress --cpu 2`), quan sát `docker stats` và đọc trực tiếp `cpu.stat` trong cgroup để xác nhận container có bị "throttle" (giới hạn) đúng như cấu hình không.
3. Chạy đồng thời nhiều container stress CPU/RAM tới mức tổng vượt quá tài nguyên thật của máy lab, quan sát sự khác biệt giữa "một container chạm limit riêng của nó" và "host thật sự quá tải" bằng cách đối chiếu `docker stats` với `vmstat`/`top` chạy song song.

## Tiêu chí hoàn thành

- Với mỗi lab, viết lại vào `notes.md` đúng chuỗi lệnh bạn đã dùng để đi từ triệu chứng tới nguyên nhân gốc, theo đúng thứ tự bạn thực sự đã làm (kể cả những bước sai/không cần thiết bạn từng thử) — mục tiêu là luyện phản xạ chẩn đoán có hệ thống, không chỉ ghi lại đáp án cuối cùng.
- Tự giải thích được bằng lời của mình sự khác biệt giữa `Exited` và `Restarting`, giữa lỗi permission và lỗi dung lượng, giữa OOM do limit sai và do host quá tải — không tra lại tài liệu khi giải thích.
- Sẵn sàng đối chiếu kết quả với [[06-Troubleshooting]] sau khi đã tự làm xong, không đọc trước.
