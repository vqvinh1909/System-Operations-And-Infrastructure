# Scripts — Module 19: Troubleshooting

Thư mục này dùng để lưu các script bạn tự viết liên quan tới module này, ví dụ:

- Script chạy nhanh toàn bộ "Checklist chẩn đoán nhanh" ở cuối [[../06-Troubleshooting|06-Troubleshooting.md]] cho một container chỉ định, in kết quả ra một báo cáo ngắn gọn.
- Script định kỳ (cron) kiểm tra `dmesg` tìm dấu hiệu OOM Killer mới xuất hiện, gửi cảnh báo ngay khi phát hiện.
- Script tổng hợp `docker system df` + dung lượng thật trên đĩa, cảnh báo sớm trước khi đĩa đầy.

Đặt tên file rõ ràng theo mục đích, ví dụ `quick-diagnose.sh`, `watch-oom-killer.sh`, `check-disk-usage.sh`.
