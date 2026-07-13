# Scripts — Module 20: Enterprise Project

Thư mục này dùng để lưu các script vận hành hỗ trợ cho toàn bộ đồ án, tách riêng khỏi mã nguồn triển khai chính trong `labs/`, ví dụ:

- Script kiểm tra sức khỏe tổng thể toàn bộ 5 VM cùng lúc (health check tổng hợp, gọi lần lượt từng node).
- Script tự động hóa việc tái tạo (từ đầu) một trong 20 tình huống sự cố ở [[../06-Troubleshooting|06-Troubleshooting.md]] trên môi trường lab, phục vụ luyện tập lại nhiều lần.
- Script tổng hợp báo cáo nhanh trước khi bàn giao/demo dự án (trạng thái từng VM, dung lượng backup gần nhất, kết quả Docker Bench for Security gần nhất).

Đặt tên file rõ ràng theo mục đích, ví dụ `healthcheck-all-nodes.sh`, `simulate-scenario-06.sh`, `generate-project-report.sh`.
