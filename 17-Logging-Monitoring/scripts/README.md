# Scripts — Module 17: Logging & Monitoring

Thư mục này dùng để lưu các script bạn tự viết liên quan tới module này, ví dụ:

- Script kiểm tra dung lượng log của tất cả container trên host và cảnh báo khi vượt ngưỡng.
- Script gọi Prometheus API (`/api/v1/query`) để lấy nhanh một chỉ số mà không cần mở UI.
- Script gọi Loki API (`/loki/api/v1/query_range`) để export log theo khung thời gian phục vụ điều tra sự cố.

Đặt tên file rõ ràng theo mục đích, ví dụ `check-container-log-size.sh`, `query-prometheus.sh`, `export-loki-range.sh`.
