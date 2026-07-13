---
title: "05 - Labs"
module: 0
tags: [ansible, sysops-infra, module-00, labs]
---

# 05 — Labs thực hành

> Lưu code/kết quả lab vào [[labs/README|labs/]]. Module này chưa dùng Ansible — mục tiêu là làm quen tư duy IaC bằng công cụ đã biết (bash, git).

## Lab 1 (Basic) — So sánh Traditional vs Idempotent script

**Chuẩn bị:** 1 VM Linux bất kỳ (dùng lại VM từ khóa Linux Sysadmin).

**Yêu cầu:**
1. Viết script `setup_v1.sh` cấu hình một "web server giả lập" theo kiểu **không idempotent**: tạo user `webapp`, tạo thư mục `/opt/webapp`, thêm dòng vào `/etc/hosts`, mỗi bước dùng lệnh raw không kiểm tra tồn tại trước (`useradd`, `mkdir`, `echo >>`).
2. Chạy `setup_v1.sh` hai lần liên tiếp. Ghi lại toàn bộ lỗi/hiện tượng xảy ra ở lần chạy thứ hai (log output).
3. Viết lại thành `setup_v2.sh` — **idempotent hóa** từng bước (dùng `id webapp &>/dev/null ||`, `mkdir -p`, `grep -qxF ... || echo ...`).
4. Chạy `setup_v2.sh` năm lần liên tiếp, chứng minh không có lỗi và trạng thái cuối cùng luôn giống nhau (dùng `diff` so sánh output của lần 1 và lần 5).

**Sản phẩm nộp:** 2 file script, 1 file log ghi lại kết quả chạy, 1 đoạn nhận xét ngắn (bạn quan sát được gì).

## Lab 2 (Intermediate) — Mô phỏng Infrastructure Lifecycle bằng Git

**Yêu cầu:**
1. Tạo repo Git `infra-lab/` theo cấu trúc ở [[04-Commands|04 - Commands]] mục 3.
2. Tạo một file `web-server-spec.md` mô tả **trạng thái mong muốn** (không phải bước thực hiện) của một web server: OS, package cần cài, port cần mở, user cần có — viết theo phong cách **Declarative** ("nginx phải được cài, state=present") chứ không phải **Imperative** ("chạy `apt install nginx`").
3. Tạo nhánh `feature/initial-spec`, commit file, viết commit message chuẩn (dạng `feat: ...`).
4. Tự đóng vai "reviewer" — viết một đoạn review comment (dạng file `REVIEW.md`) chỉ ra ít nhất 1 điểm chưa rõ ràng hoặc thiếu trong spec ban đầu, rồi sửa lại và commit tiếp.
5. Merge vào `main`, xem lại `git log --oneline --graph` để thấy lịch sử thay đổi — đối chiếu với vấn đề "No Audit Trail" của mô hình Traditional đã học.

**Sản phẩm nộp:** link/tên repo local, output của `git log --oneline --graph --all`.

## Lab 3 (Mini Project) — Thiết kế Enterprise IaC Architecture cho một tình huống thật

**Tình huống:** Một công ty thương mại điện tử vừa quyết định chuyển từ vận hành thủ công sang IaC. Hạ tầng hiện tại gồm: 3 web server, 1 database server, 1 load balancer, tất cả trên VM on-premise. Đội ngũ có 4 kỹ sư, dùng GitLab nội bộ.

**Yêu cầu:** Viết một tài liệu đề xuất (`proposal.md`, 1-2 trang) gồm:
1. Vẽ sơ đồ ASCII kiến trúc IaC đề xuất cho công ty này (dựa theo mẫu ở [[03-Architecture|03 - Architecture]] Sơ đồ 3, điều chỉnh cho đúng số lượng server của tình huống).
2. Liệt kê rõ: ai được quyền merge vào nhánh chính, secret (mật khẩu DB) sẽ được lưu ở đâu (chưa cần chi tiết kỹ thuật Vault — chỉ cần nêu nguyên tắc "không hardcode").
3. Đề xuất quy trình xử lý khi phát hiện configuration drift (server bị sửa tay ngoài quy trình Git).
4. Giải thích ngắn gọn cho Product Manager (không rành kỹ thuật) — bằng ngôn ngữ phi kỹ thuật — vì sao việc chuyển đổi này đáng giá thời gian đầu tư ban đầu.

**Sản phẩm nộp:** file `proposal.md` hoàn chỉnh, có sơ đồ ASCII tự đo bằng script như hướng dẫn ở Module 00 README.
