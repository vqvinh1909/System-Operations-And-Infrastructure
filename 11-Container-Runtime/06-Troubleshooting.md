---
title: "06 - Troubleshooting"
module: 11
tags: [docker, sysops-infra, module-11, troubleshooting]
---

# 06 - Troubleshooting: Container Runtime

## Sự cố 1: Restart loop — container restart liên tục, alert dồn dập lúc nửa đêm

**Chẩn đoán theo thứ tự**:
```bash
docker inspect <container> --format '{{.RestartCount}}'
docker inspect <container> --format '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}}'
docker logs --tail 200 <container>
docker events --filter "container=<container>" --since 1h
```

**Nguyên nhân thường gặp**: (1) OOM kill lặp lại do `--memory` set quá thấp so với nhu cầu thực (`OOMKilled: true`); (2) lỗi ứng dụng thật (exception khi khởi động, sai config, không kết nối được dependency như database) — exit code khác 0, không phải OOM; (3) healthcheck fail liên tục khiến orchestrator bên ngoài chủ động restart.

**Khắc phục**: xử lý đúng nguyên nhân gốc theo log/exit code tìm được — không chỉ tăng `--memory` hoặc đổi restart policy để "che" triệu chứng. Nếu là lỗi kết nối dependency lúc khởi động, cân nhắc thêm cơ chế retry với backoff trong ứng dụng và `--restart=on-failure:N` thay vì `always` để tránh restart loop vô hạn khi lỗi thực sự không tự khỏi.

## Sự cố 2: Ổ đĩa host đầy do log container không giới hạn

**Chẩn đoán**:
```bash
docker system df -v
du -sh /var/lib/docker/containers/*/  | sort -rh | head -10
```

**Nguyên nhân**: logging driver `json-file` mặc định không giới hạn kích thước — một container ghi log liên tục (vòng lặp lỗi, log level DEBUG quên tắt) trong nhiều tuần/tháng phình file log lên hàng chục GB.

**Khắc phục ngay**: tìm container thủ phạm, cấu hình lại `--log-opt max-size`/`max-file` (yêu cầu tạo lại container vì log-opt không đổi được bằng `docker update`), hoặc tạm thời `truncate -s 0 <log-path>` để giải phóng gấp dung lượng (chỉ nên dùng khi khẩn cấp, mất log cũ). Khắc phục lâu dài: đặt giới hạn log mặc định ở `/etc/docker/daemon.json` để áp dụng cho mọi container mới, tránh phải nhớ cấu hình thủ công từng lần.

## Sự cố 3: Container "treo" khi `docker stop`, luôn mất đúng 10 giây rồi mới dừng

**Chẩn đoán**:
```bash
docker exec <container> ps aux
# Kiem tra PID 1 co phai la /bin/sh -c thay vi ung dung that khong
```

**Nguyên nhân**: `CMD`/`ENTRYPOINT` dùng shell form khiến ứng dụng không phải PID 1 thật, hoặc ứng dụng là PID 1 nhưng không tự đăng ký handler xử lý `SIGTERM` — kernel không áp dụng hành vi mặc định cho PID 1 nếu không có handler, container luôn phải đợi hết grace period rồi mới bị `SIGKILL`.

**Khắc phục**: sửa Dockerfile dùng exec form, thêm xử lý `SIGTERM` trong code ứng dụng (hầu hết ngôn ngữ/framework hiện đại có sẵn cơ chế này), hoặc chạy container với `--init` để `tini` forward signal đúng chuẩn.

## Sự cố 4: `docker system prune -a --volumes` xóa nhầm dữ liệu quan trọng trên production

**Tình huống thực tế đã xảy ra**: kỹ sư chạy `docker system prune -a --volumes` để "dọn dẹp cho nhanh" trước khi rời ca, không kiểm tra kỹ danh sách trước — xóa mất volume chứa dữ liệu backup tạm thời của một job đã dừng nhưng dữ liệu chưa kịp archive đi nơi khác.

**Bài học và phòng ngừa**:
- Luôn chạy `docker system df -v` và `docker volume ls` trước, xác nhận rõ danh sách sẽ bị xóa.
- Không bao giờ dùng `--volumes` trong script tự động hóa chạy định kỳ trên production mà không có bước xác nhận thủ công hoặc whitelist rõ ràng.
- Ưu tiên dọn dẹp có chọn lọc (`docker container prune`, `docker image prune -a` riêng biệt) thay vì `docker system prune -a --volumes` "dọn hết một lần" trên môi trường có dữ liệu quan trọng.

## Sự cố 5: Container CPU bị throttle dù `docker stats` báo %CPU thấp hơn giới hạn `--cpus`

**Chẩn đoán**:
```bash
CID=$(docker inspect --format '{{.Id}}' <container>)
cat /sys/fs/cgroup/system.slice/docker-${CID}.scope/cpu.stat
# Xem nr_throttled va throttled_usec
```

**Nguyên nhân**: `--cpus` hoạt động theo cơ chế quota/period (mặc định period 100ms) — ứng dụng có thể bị throttle trong các khoảng burst ngắn dù %CPU trung bình đo bởi `docker stats` (lấy mẫu theo giây) trông có vẻ thấp. Đây là hiện tượng micro-throttling, thường gặp với ứng dụng đa luồng có burst CPU ngắn (ví dụ JIT compile, garbage collection).

**Khắc phục**: tăng `--cpus` nếu có dư tài nguyên host, hoặc chuyển sang `--cpu-shares` nếu chỉ cần ưu tiên tương đối thay vì giới hạn cứng theo chu kỳ, hoặc điều chỉnh ứng dụng giảm số lượng thread song song để tránh burst quá ngắn hạn nhưng cao đỉnh.

## Sự cố 6: `docker exec` vào container để chạy lệnh chẩn đoán nhưng vô tình làm ảnh hưởng service đang chạy

**Nguyên nhân**: nhầm dùng `docker attach` thay vì `docker exec`, sau đó nhấn `Ctrl+C` để "thoát" — tín hiệu `SIGINT` truyền thẳng tới tiến trình chính (PID 1), có thể làm dừng luôn service đang phục vụ traffic thật.

**Khắc phục/phòng ngừa**: luôn dùng `docker exec -it <container> sh` để debug (tạo tiến trình độc lập, không đụng chạm PID 1); nếu buộc phải dùng `docker attach`, luôn thoát bằng tổ hợp phím detach đúng chuẩn `Ctrl+P` rồi `Ctrl+Q`, tuyệt đối không dùng `Ctrl+C`.
