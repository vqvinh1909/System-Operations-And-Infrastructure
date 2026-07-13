---
title: "06 - Troubleshooting"
module: 18
tags: [docker, sysops-infra, module-18, security, troubleshooting]
---

# 06 — Troubleshooting

## Sự cố 1: Container không bind được port sau khi chuyển sang Rootless Docker

**Triệu chứng:** `docker run -p 80:80 ...` báo lỗi hoặc port không thể truy cập, dù lệnh tương tự chạy bình thường trên Docker rootful.

**Nguyên nhân gốc:** daemon rootless không có quyền root để bind port < 1024 theo cách bridge network thông thường — đây là hệ quả trực tiếp của chính cơ chế bảo mật đã học ở [[02-Theory]], không phải lỗi cấu hình.

**Khắc phục:**
```bash
# Cach 1: dung port >= 1024, roi dat reverse proxy/load balancer (chay rootful hoac
# co quyen phu hop) o phia truoc de forward tu port 80 that
docker run -d -p 8080:80 nginx:1.27

# Cach 2 (Linux, tuy distro): cho phep user thuong bind port thap hon threshold mac dinh
sudo sysctl net.ipv4.ip_unprivileged_port_start=80
```

## Sự cố 2: Ứng dụng lỗi ngay khi khởi động sau khi thêm `--cap-drop=ALL`

**Triệu chứng:** container thoát ngay (`Exited`) hoặc log báo `Operation not permitted`/`Permission denied` ngay ở bước khởi động, dù image không đổi.

**Chẩn đoán:**
```bash
docker logs <container>
docker run --rm -it --cap-drop=ALL <image> sh   # chay tuong tac de thu truc tiep
```

**Nguyên nhân thường gặp:** entrypoint/init script của image thực hiện một hành động cần capability cụ thể (ví dụ đổi owner file cấu hình bằng `chown` lúc khởi động — cần `CAP_CHOWN`; hoặc bind port thấp bên trong container — cần `CAP_NET_BIND_SERVICE`).

**Khắc phục:** không revert về mặc định (bỏ hẳn `--cap-drop=ALL`) — thay vào đó xác định đúng capability còn thiếu bằng cách đọc kỹ thông báo lỗi (thường ghi rõ syscall/hành động bị từ chối), thêm đúng `--cap-add` tương ứng, giữ nguyên phần còn lại đã bị drop.

## Sự cố 3: Container bị "treo"/lỗi lạ sau khi áp seccomp profile tùy chỉnh

**Triệu chứng:** ứng dụng chạy chậm bất thường, treo ở một thao tác cụ thể, hoặc crash với lỗi không rõ ràng (không phải lỗi ứng dụng logic bình thường) sau khi áp `--security-opt seccomp=<file custom>`.

**Chẩn đoán:**
```bash
# Kiem tra kernel/audit log tim dong bi seccomp chan (thuong ghi nhan qua audit hoac dmesg)
sudo dmesg | grep -i seccomp
sudo journalctl -k | grep -i seccomp
```

**Nguyên nhân gốc:** profile tùy chỉnh thiếu một syscall mà ứng dụng thật sự cần (khác với những gì bạn liệt kê thủ công khi viết profile) — ứng dụng không nhận được lỗi rõ ràng như capability, mà tiến trình gọi syscall bị chặn thường nhận tín hiệu bị kill (`SIGSYS`) ngay lập tức, dễ nhầm là "app tự crash".

**Khắc phục:** quay lại seccomp profile mặc định của Docker để xác nhận vấn đề đúng là do seccomp (không phải lỗi khác), sau đó bổ sung dần từng syscall còn thiếu vào profile tùy chỉnh, test lại sau mỗi lần thêm — tránh thêm hàng loạt syscall một lúc vì sẽ làm mất mục đích siết chặt ban đầu.

## Sự cố 4: AppArmor chặn container ghi/đọc một đường dẫn tưởng chừng hợp lệ

**Triệu chứng:** ứng dụng báo lỗi `Permission denied` khi ghi vào một thư mục **đã được mount đúng** bằng `-v`, dù quyền Unix (owner/group/mode) của thư mục đó hoàn toàn hợp lệ.

**Chẩn đoán:**
```bash
sudo journalctl -k | grep -i apparmor | grep -i DENIED
sudo dmesg | grep -i apparmor
```

**Nguyên nhân gốc:** đây là điểm dễ gây nhầm lẫn nhất khi mới làm quen AppArmor trong container — quyền Unix cho phép (`rwx` đúng), nhưng **AppArmor profile `docker-default` áp thêm một lớp kiểm soát riêng, độc lập với quyền Unix**, và có thể từ chối truy cập dù quyền Unix hợp lệ.

**Khắc phục:** xác nhận đúng nguyên nhân qua log AppArmor (không phải đoán), sau đó hoặc điều chỉnh mount point về đường dẫn được profile mặc định cho phép, hoặc (nếu thật sự cần) viết một AppArmor profile tùy chỉnh nới lỏng đúng đường dẫn cần thiết — tuyệt đối tránh phản xạ tắt hẳn AppArmor (`unconfined`) chỉ để "cho qua chuyện", vì điều này bỏ toàn bộ lớp bảo vệ chứ không chỉ riêng đường dẫn đang gặp vấn đề.

## Sự cố 5: Docker Scout / Trivy báo hàng trăm CVE, không biết bắt đầu xử lý từ đâu

**Triệu chứng:** kết quả scan một image base cũ trả về số lượng CVE lớn đến mức gây choáng ngợp, dễ dẫn tới việc bỏ qua toàn bộ báo cáo vì "không thể xử lý hết".

**Cách tiếp cận đúng:**
1. Lọc trước theo mức độ nghiêm trọng — chỉ tập trung `Critical` và `High` trước (`trivy image --severity CRITICAL,HIGH`), phần lớn số lượng CVE khổng lồ thường nằm ở mức `Low`/`Medium` ít ảnh hưởng thực tế ngay lập tức.
2. Kiểm tra xem CVE đó **đã có bản vá (fixed version)** hay chưa — nhiều công cụ scan phân biệt rõ "có fix, chỉ cần update package" và "chưa có fix, cần đánh giá rủi ro/tìm giải pháp thay thế". Ưu tiên xử lý nhóm đã có fix trước vì chi phí thấp, hiệu quả cao.
3. Đánh giá base image — thường phần lớn CVE nằm trong base image (OS packages), không phải code ứng dụng. Chuyển sang base image tối giản hơn (ví dụ Alpine, distroless nếu phù hợp) thường giảm số lượng CVE nhiều hơn việc cố vá từng CVE lẻ tẻ.

## Sự cố 6: Docker Bench for Security báo `[WARN]` cho mục không áp dụng được với hạ tầng của bạn

**Triệu chứng:** một số mục cảnh báo liên quan tới Docker Swarm, hoặc cấu hình cụ thể bạn không hề dùng.

**Xử lý đúng:** không cần "làm sạch" toàn bộ output một cách máy móc. Đọc mô tả đầy đủ của từng `[WARN]` (có trong output hoặc tài liệu CIS Docker Benchmark), xác định mục nào thật sự áp dụng cho kiến trúc bạn đang vận hành, ghi chú lại lý do bỏ qua những mục không áp dụng (phục vụ audit sau này nếu có ai hỏi lại), và chỉ dồn công sức khắc phục những mục thật sự liên quan tới rủi ro của hệ thống bạn.
