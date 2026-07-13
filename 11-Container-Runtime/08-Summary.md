---
title: "08 - Summary"
module: 11
tags: [docker, sysops-infra, module-11, summary]
---

# 08 - Summary: Container Runtime

## Những gì đã học

- **Vòng đời chi tiết**: created → running → (paused) → exited → removed, cùng ý nghĩa vận hành của `create`/`start`/`run`/`stop`/`kill`/`pause`/`unpause`/`restart`/`rm`.
- **Signal handling**: PID 1 trong container chịu quy tắc đặc biệt của kernel — không tự áp dụng hành vi mặc định cho signal nếu không có handler. `--init`/`tini` giải quyết cả vấn đề signal forwarding lẫn zombie reaping.
- **Resource limit**: `--memory`/`--memory-reservation`/`--cpus`/`--cpu-shares`/`--cpuset-cpus` map trực tiếp xuống cgroups v2 (`memory.max`, `memory.low`, `cpu.max`, `cpu.weight`, `cpuset.cpus`) — không có "phép thuật" nào ngoài cơ chế Module 08 đã học.
- **OOM Kill**: vượt `--memory` → kernel OOM Killer → SIGKILL → exit code 137 + `OOMKilled: true`. Luôn xác nhận bằng `docker inspect`, không suy đoán từ exit code.
- **Restart Policy**: `no`/`on-failure[:N]`/`always`/`unless-stopped` — chọn đúng theo loại workload, hiểu rõ đây không phải phép màu, không sửa nguyên nhân gốc và không thay thế healthcheck.
- **Logging**: mặc định `json-file` không giới hạn — bắt buộc cấu hình `max-size`/`max-file` per-container hoặc ở cấp daemon để tránh disk đầy.
- **Debug an toàn**: `docker exec` (tiến trình mới, an toàn) ưu tiên hơn `docker attach` (kết nối trực tiếp PID 1, rủi ro).
- **Bộ công cụ vận hành bắt buộc**: `docker events` (sự kiện realtime), `docker stats` (tài nguyên realtime), `docker system df` (dung lượng đang chiếm), `docker system prune` (dọn dẹp có kiểm soát, cẩn trọng `-a --volumes`).

## Checklist tự đánh giá

- [ ] Phân biệt chính xác `stop`/`kill`/`pause` và biết khi nào dùng lệnh nào.
- [ ] Giải thích được vì sao PID 1 trong container cần xử lý signal khác biệt, và khi nào cần `--init`.
- [ ] Cấu hình được resource limit và đọc đúng bằng chứng OOM kill qua `docker inspect`/`dmesg`.
- [ ] Chọn đúng restart policy cho từng loại workload (batch job, worker, service 24/7, web app thông thường).
- [ ] Cấu hình log rotation tránh disk đầy, cả per-container lẫn cấp daemon.
- [ ] Dùng thành thạo `docker events`/`docker stats`/`docker system df`/`docker system prune` trong một kịch bản giám sát/dọn dẹp thực tế.

## Liên kết tới Module tiếp theo

Module này hoàn thiện kỹ năng vận hành một container **đơn lẻ**. Nhưng ứng dụng thực tế hiếm khi chỉ có một container độc lập — chúng cần giao tiếp với nhau (web gọi database, service gọi service). [[../12-Networking/README|Module 12 - Networking]] sẽ trả lời câu hỏi: container "nói chuyện" với nhau và với thế giới bên ngoài như thế nào — bridge network, host network, DNS nội bộ, port mapping. Kiến thức Network Namespace đã học ở Module 08 sẽ được áp dụng trực tiếp để hiểu cách Docker dựng network driver bên dưới.

> [!tip] Ghi nhớ cốt lõi
> Vận hành container không phải chỉ "chạy được là xong" — nó là chuỗi quyết định có chủ đích: giới hạn tài nguyên đúng mức, chọn restart policy đúng loại workload, giới hạn log trước khi nó làm đầy disk, và luôn có cách debug an toàn không làm gián đoạn service đang chạy thật. Đây chính là ranh giới giữa "biết dùng Docker" và "vận hành Docker ở production".
