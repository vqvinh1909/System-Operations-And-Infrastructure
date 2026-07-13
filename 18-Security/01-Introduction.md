---
title: "01 - Introduction"
module: 18
tags: [docker, sysops-infra, module-18, security]
---

# 01 — Giới thiệu Module: Security

## Tình huống mở đầu

Một dev trong công ty bạn deploy một container chạy ứng dụng web nội bộ, dùng đúng lệnh cơ bản đã học ở Part II: `docker run -d -p 8080:80 internal-app:v1`. Ứng dụng chạy tốt suốt 3 tháng. Rồi một ngày, do một lỗ hổng trong chính thư viện bên trong image (CVE đã tồn tại từ lúc build nhưng không ai từng scan qua), kẻ tấn công khai thác được remote code execution ngay trong container đó. Câu hỏi sống còn lúc này: **kẻ tấn công đang có quyền gì bên trong container đó, và từ container đó có thể leo thang (escalate) ra tới đâu?**

Nếu container đó chạy dưới Docker daemon mặc định (rootful), tiến trình bên trong container về bản chất vẫn là tiến trình con của một daemon chạy bằng UID 0 (root) trên host. Nếu container không bị giới hạn capability, không có seccomp/AppArmor chặn bớt, một lỗ hổng kernel kết hợp với cấu hình lỏng lẻo có thể biến "chiếm được 1 container" thành "chiếm được toàn bộ host" — kịch bản gọi là **container breakout**.

Module này dạy bạn cách giảm thiểu triệt để hậu quả của kịch bản trên, bằng nhiều lớp phòng thủ độc lập chồng lên nhau — để dù một lớp bị vượt qua, các lớp còn lại vẫn ngăn được thiệt hại lan rộng.

## Vì sao module này quan trọng với vai trò Sysadmin/DevOps

Ở khóa Linux Sysadmin, bạn đã học Security Hardening ở mức hệ điều hành: giảm quyền root, cấu hình SELinux/AppArmor, đóng cổng không cần thiết, nguyên tắc least privilege. Container không miễn nhiễm với những nguyên tắc đó — thực ra nó cần áp dụng **nghiêm ngặt hơn**, vì một host chạy Docker thường co-locate (đặt chung) hàng chục container từ nhiều nhóm/dự án khác nhau trên cùng một kernel. Một container bị chiếm mà không bị giới hạn đúng cách không chỉ ảnh hưởng riêng nó — nó đe dọa mọi container khác và chính host.

Trong công việc thực tế, sysadmin/DevOps chịu trách nhiệm về bảo mật Docker thường phải:

- Quyết định khi nào bắt buộc dùng Rootless Docker, khi nào rootful vẫn chấp nhận được (có đánh đổi).
- Viết policy `--cap-drop`/`--cap-add` chuẩn cho từng loại ứng dụng, không copy-paste "chạy full quyền cho nhanh".
- Đưa image scanning thành một **gate bắt buộc** trong pipeline CI/CD, không phải bước tùy chọn.
- Chạy định kỳ Docker Bench for Security để tự kiểm tra hệ thống có lệch khỏi CIS Benchmark hay không.

## Mục tiêu học tập

Sau module này, bạn sẽ:

1. Cài đặt và vận hành được Docker ở chế độ Rootless, hiểu rõ đánh đổi so với rootful.
2. Viết đúng cấu hình `--cap-drop=ALL` kết hợp `--cap-add` tối thiểu cần thiết cho một ứng dụng cụ thể.
3. Giải thích và áp dụng được Seccomp profile, AppArmor profile cho container, liên hệ lại kiến thức AppArmor đã học ở khóa trước.
4. Scan được image tìm lỗ hổng CVE bằng công cụ thực tế, đọc hiểu kết quả và ưu tiên xử lý theo mức độ nghiêm trọng.
5. Chạy Docker Bench for Security, đọc hiểu kết quả và biết mục nào cần ưu tiên khắc phục trước.

## Kiến thức nền cần có trước

- Đã hoàn thành Module 17 (Logging & Monitoring).
- Đã học Security Hardening (least privilege, SELinux/AppArmor cơ bản, giảm quyền root) ở khóa Linux Sysadmin — module này liên hệ trực tiếp, không dạy lại từ đầu khái niệm AppArmor là gì.
- Hiểu khái niệm namespace, cgroup từ Part II (Container Technology) — nền tảng để hiểu vì sao Rootless Docker hoạt động được.
