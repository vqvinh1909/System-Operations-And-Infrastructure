# Scripts — Module 18: Security

Thư mục này dùng để lưu các script bạn tự viết liên quan tới module này, ví dụ:

- Script quét hàng loạt image đang chạy trên host bằng Trivy, tổng hợp báo cáo CVE mức Critical/High.
- Script kiểm tra nhanh capability/AppArmor profile của toàn bộ container đang chạy (`docker inspect` hàng loạt).
- Script chạy Docker Bench for Security định kỳ (cron) và gửi cảnh báo khi có `[WARN]` mới xuất hiện so với lần chạy trước.

Đặt tên file rõ ràng theo mục đích, ví dụ `scan-all-running-images.sh`, `audit-container-security-opts.sh`, `run-docker-bench-cron.sh`.
