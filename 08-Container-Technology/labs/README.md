---
title: "Labs - Module 08"
module: 8
tags: [docker, sysops-infra, module-08, labs]
---

# Labs - Module 08: Container Technology

Thư mục này chứa các file bạn tự tạo ra trong quá trình làm lab ở [[../05-Labs|05-Labs.md]].

## Môi trường

- 1 VM Linux (khuyến nghị Ubuntu 24.04, có cgroups v2 mặc định).
- Quyền `sudo`.
- Gói `util-linux` (có sẵn trên hầu hết distro) cho `unshare`, `nsenter`, `lsns`.
- Docker Engine đã cài (chỉ cần cho Lab 3 - mô phỏng `docker exec`; việc cài đặt chi tiết học ở Module 09).
- Tùy chọn: `stress-ng` để tạo tải CPU/RAM thử nghiệm cgroup (`sudo apt install stress-ng`).

## File dự kiến sinh ra trong thư mục này

| File | Sinh ra ở Lab | Nội dung |
|---|---|---|
| `namespace-survey-notes.txt` | Lab 1 | Ghi chú inode number namespace của các tiến trình đã khảo sát |
| `unshare-notes.md` | Lab 2 | Bảng namespace đã tạo, lệnh kiểm chứng, kết quả quan sát |
| `mini-container.sh` | Lab 5 | Script tự động hóa dựng mini container bằng `unshare` + `chroot` + cgroups |

## Lưu ý an toàn khi làm lab

- Không chạy các lệnh `unshare`/thao tác cgroup trực tiếp trên server production — dùng VM lab riêng.
- Lệnh cgroup thao tác trực tiếp `/sys/fs/cgroup` có thể ảnh hưởng tiến trình hệ thống nếu gõ nhầm đường dẫn — luôn kiểm tra lại đường dẫn cgroup trước khi `echo ... | tee`.
- Dọn dẹp cgroup/namespace tạm sau mỗi lab (`rmdir` cgroup, thoát shell `unshare`) để tránh rác tồn đọng trên hệ thống lab.
