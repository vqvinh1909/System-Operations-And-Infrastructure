---
title: "Labs - Module 09"
module: 9
tags: [docker, sysops-infra, module-09, labs]
---

# Labs - Module 09: Docker Engine

Thư mục lưu kết quả thực hành từ [[../05-Labs|05-Labs.md]].

## Môi trường

- VM Linux đã cài Docker Engine 29.x, quyền `sudo`.
- Truy cập internet để pull image `nginx`, `busybox`, `python:3.12-slim` từ Docker Hub (hoặc registry mirror nội bộ nếu đã cấu hình).

## File dự kiến sinh ra trong thư mục này

| File | Sinh ra ở Lab | Nội dung |
|---|---|---|
| `daemon-audit.md` | Lab 1 | Bảng thông tin daemon: storage driver, cgroup driver/version, socket permission |
| `daemon.json` | Lab 3 | Bản cấu hình daemon đã kiểm thử thành công |
| `signal-handler-demo.py` | Lab 4 | Script demo bắt SIGTERM để so sánh `stop` vs `kill` |
| `security-finding.md` | Lab 5 | Báo cáo rủi ro bảo mật nhóm `docker`, viết theo văn phong security finding thực tế |

## Lưu ý an toàn

- Lab 5 (mô phỏng chiếm quyền root qua nhóm `docker`) **chỉ thực hiện trên VM lab dùng riêng**, tuyệt đối không thử trên server có dữ liệu thật hoặc server dùng chung với người khác.
- Sau khi hoàn thành Lab 5, nhớ gỡ user thử nghiệm khỏi nhóm `docker` và xóa user để không để lại lỗ hổng trên VM lab.
