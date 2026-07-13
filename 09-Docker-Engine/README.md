---
title: "Module 09 - Docker Engine"
tags: [docker, sysops-infra, module-09, engine, architecture]
module: 9
part: "II - Container Fundamentals"
difficulty: Intermediate
status: draft
created: 2026-07-12
prerequisites: ["[[../08-Container-Technology/README|Module 08]]"]
next: "[[../10-Docker-Images/README|Module 10]]"
---

# Module 09 - Docker Engine

## Vì sao module này quan trọng

Ở Module 08, bạn đã hiểu container là gì ở tầng kernel: namespace cô lập, cgroup giới hạn tài nguyên, OverlayFS ghép layer thành filesystem. Nhưng khi bạn gõ `docker run nginx`, không phải kernel tự nhiên biết phải làm những việc đó — có cả một chuỗi phần mềm đứng giữa lệnh bạn gõ và namespace/cgroup thực sự được tạo ra. Chuỗi đó chính là **Docker Engine**.

Hiểu rõ Docker Engine không phải bài tập lý thuyết suông. Trong công việc sysadmin/DevOps thực tế, bạn sẽ gặp:

- Daemon Docker không start được sau khi reboot server — bạn cần biết `dockerd` đọc cấu hình từ đâu, log ở đâu để chẩn đoán.
- Lập trình viên báo "container tự nhiên biến mất" — bạn cần phân biệt được container bị `docker rm`, bị OOM killer giết, hay do `containerd-shim` chết theo `dockerd` (thực ra không nên xảy ra, và biết tại sao nó không xảy ra chính là hiểu kiến trúc).
- Onboard nhân viên mới, thêm họ vào nhóm `docker` — bạn phải biết đây gần như tương đương cấp quyền root, để không biến một quyết định "cho tiện" thành lỗ hổng bảo mật.
- Container thoát ra với exit code 137 lúc nửa đêm — bạn cần phân biệt `docker stop` (SIGTERM) và bị kill cưỡng bức (SIGKILL) để tìm đúng nguyên nhân (OOM, healthcheck, hay do admin khác thao tác).

Module này xây nền tảng kiến trúc để bạn không chỉ "gõ lệnh cho chạy" mà thực sự hiểu điều gì xảy ra bên dưới — đây là ranh giới giữa người dùng Docker và người vận hành Docker ở mức production.

## Cấu trúc nội dung

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Docker Engine là gì, mục tiêu học tập |
| [[02-Theory]] | Lý thuyết chi tiết: kiến trúc client-server, các thành phần, Internal Working |
| [[03-Architecture]] | Sơ đồ ASCII: luồng gọi từ CLI tới container process, vòng đời container, rủi ro socket |
| [[04-Commands]] | Cú pháp lệnh, ví dụ `daemon.json`, các lệnh quản lý container |
| [[05-Labs]] | Bài lab thực hành từ cơ bản đến nâng cao |
| [[06-Troubleshooting]] | Lỗi thường gặp và cách chẩn đoán |
| [[07-Interview]] | Câu hỏi phỏng vấn 3 cấp độ |
| [[08-Summary]] | Tóm tắt và cầu nối sang Module 10 |
| [[labs/README|labs/]] | Nơi lưu code/config thực hành |
| [[scripts/README|scripts/]] | Nơi lưu script tự viết |

## Điều kiện tiên quyết

- Đã hoàn thành [[../08-Container-Technology/README|Module 08 - Container Technology]] (namespace, cgroup, OverlayFS, so sánh VM/container).
- Biết SSH, systemd, quản lý permission Linux (từ chương trình Linux Sysadmin 29 module đã học).
- Có Docker Engine đã cài trên máy thực hành (Linux, hoặc WSL2 trên Windows).

## Sau module này bạn sẽ

- Vẽ và giải thích được luồng: `docker CLI -> REST API/socket -> dockerd -> containerd -> containerd-shim -> runc -> container process`.
- Đọc và chỉnh sửa được `daemon.json` an toàn, biết reload đúng cách.
- Giải thích được vì sao nhóm `docker` gần như tương đương quyền root, và cách giảm thiểu rủi ro đó trong môi trường doanh nghiệp.
- Phân biệt rạch ròi các trạng thái vòng đời container và biết dùng đúng lệnh (`stop` vs `kill`, `pause` vs `stop`).
- Chẩn đoán được các lỗi vận hành phổ biến: daemon không start, permission denied trên socket, container Exited(137).

---

> [!note] Self-Review
> - Đã WebSearch xác nhận (tính đến 2026-07-12): nhánh Docker Engine hiện hành 29.x (vd 29.6.1, 6/2026); BuildKit là builder mặc định cho `docker build` trên Linux từ Docker Engine 23.0; Docker Compose CLI plugin nhánh 5.x (vd v5.3.1, 7/2026); cgroups v2 là mặc định trên distro hiện đại, driver cgroup khuyến nghị `systemd`; OCI Runtime Spec v1.3.0 (11/2025), OCI Image Spec v1.1.0 (2/2024); runc nhánh 1.5.x, containerd nhánh 2.2.x giữa 2026; kiến trúc dockerd -> containerd -> containerd-shim -> runc và vai trò shim (tránh orphan process, containerd không cần giữ process cha cho mỗi container) đã được đối chiếu với tài liệu chính thức.
> - Đã đo cả 3 sơ đồ ASCII trong [[03-Architecture]] bằng script Python đo độ dài từng dòng khung (border trên/dưới/nội dung) — kết quả toàn bộ nhóm khung đều `OK`, không còn `MISMATCH`. Lưu ý kỹ thuật: các dòng mũi tên nối khung dùng ký tự `|` và `v` được tách riêng bằng dòng trống để không bị gộp nhầm nhóm với khung chữ nhật liền kề khi đo.
