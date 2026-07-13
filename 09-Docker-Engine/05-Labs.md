---
title: "05 - Labs"
module: 9
tags: [docker, sysops-infra, module-09, labs, hands-on]
---

# 05 - Labs: Docker Engine

Môi trường: 1 VM Linux đã cài Docker Engine (29.x), quyền `sudo`. Lưu file/script tạo ra trong quá trình lab vào `labs/` (xem [[labs/README|labs/README.md]]).

## Lab 1 (Basic): Khảo sát kiến trúc daemon đang chạy

1. Chạy `systemctl status docker`, ghi lại PID của `dockerd`.
2. Chạy `ps -ef | grep -E 'dockerd|containerd'` — xác nhận `containerd` là tiến trình độc lập, không phải con trực tiếp của `dockerd`.
3. Chạy `docker info` — ghi lại: Storage Driver, Cgroup Driver, Cgroup Version, Server Version.
4. Chạy `ls -l /var/run/docker.sock` — ghi lại owner, group, permission mode.
5. Chạy `journalctl -u docker.service -n 50 --no-pager` — đọc log daemon lúc khởi động, xác định daemon đọc `daemon.json` từ đâu.

**Kết quả cần đạt**: một bảng ghi lại đầy đủ thông tin trên, kèm giải thích ngắn ý nghĩa từng dòng.

## Lab 2 (Basic): Quan sát tiến trình shim và tính độc lập với dockerd

1. Chạy `docker run -d --name lab-nginx nginx:1.27-alpine`.
2. Chạy `ps -ef | grep containerd-shim` — ghi lại PID shim ứng với container này.
3. Chạy `docker inspect lab-nginx --format '{{.State.Pid}}'` — so sánh PID này (PID container process trên host) với PID shim vừa tìm — xác nhận shim là cha trực tiếp.
4. Chạy `sudo systemctl restart docker`.
5. Ngay sau khi `dockerd` khởi động lại, chạy `docker ps` — xác nhận `lab-nginx` vẫn đang `Up`, và kiểm tra lại PID shim ở bước 2 — PID đó vẫn còn sống, không đổi.
6. Dọn dẹp: `docker rm -f lab-nginx`.

**Kết quả cần đạt**: bằng chứng thực nghiệm (output lệnh trước/sau restart) chứng minh container sống sót qua việc restart `dockerd` nhờ kiến trúc shim.

## Lab 3 (Intermediate): Cấu hình `daemon.json` an toàn

1. Sao lưu cấu hình hiện tại: `sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak` (nếu file đã tồn tại; nếu chưa có, tạo mới).
2. Viết `daemon.json` với tối thiểu: `log-driver`, `log-opts` (giới hạn `max-size`, `max-file`), `exec-opts` ép cgroup driver `systemd`.
3. Kiểm tra cú pháp JSON hợp lệ trước khi áp dụng: `python3 -m json.tool /etc/docker/daemon.json` (nếu lỗi cú pháp, `dockerd` sẽ không khởi động được).
4. `sudo systemctl restart docker`, xác nhận `systemctl status docker` là `active (running)`.
5. Xác nhận cấu hình đã áp dụng: `docker info | grep -iE 'cgroup|logging'`.
6. Chạy thử một container sinh log liên tục (`docker run -d --name loggy busybox sh -c "while true; do echo log line; sleep 0.1; done"`), đợi vài phút, kiểm tra file log thực tế không vượt quá giới hạn đã cấu hình: `docker inspect loggy --format '{{.LogPath}}'` rồi `ls -lh` file đó.
7. Dọn dẹp: `docker rm -f loggy`.

**Kết quả cần đạt**: `daemon.json` hoàn chỉnh đã áp dụng thành công, cùng bằng chứng log bị giới hạn đúng dung lượng cấu hình.

## Lab 4 (Intermediate): Vòng đời container — phân biệt `stop`, `kill`, `pause`

1. Viết một script Python/Node đơn giản có bắt tín hiệu `SIGTERM` (in ra "received SIGTERM, cleaning up..." rồi `sleep 3` trước khi thoát), đóng gói chạy trong container (dùng image `python:3.12-slim` chạy trực tiếp script qua bind mount, chưa cần build image riêng — kỹ thuật build học ở Module 10).
2. Chạy container ở chế độ `-d`, thử `docker stop` — xem log bằng `docker logs` để xác nhận có in ra dòng "received SIGTERM" trước khi container dừng, đo thời gian dừng thực tế (khoảng 3 giây, đúng bằng thời gian script tự dọn dẹp).
3. Chạy lại container tương tự, lần này dùng `docker kill` — quan sát: script KHÔNG kịp in dòng log dọn dẹp, container dừng gần như ngay lập tức.
4. Chạy lại container, thử `docker pause` rồi `docker stats` trên một terminal khác — xác nhận container không xuất hiện %CPU dù tiến trình bên trong đang ở vòng lặp; `docker unpause` rồi kiểm tra lại.
5. Ghi lại bảng so sánh thời gian phản hồi và hành vi quan sát được của `stop`/`kill`/`pause`.

**Kết quả cần đạt**: bảng so sánh thực nghiệm 3 lệnh, kèm log chứng minh graceful shutdown chỉ xảy ra với `stop`.

## Lab 5 (Advanced): Mô phỏng rủi ro bảo mật nhóm `docker` trong môi trường lab cô lập

**Cảnh báo**: chỉ thực hiện trên VM lab dùng riêng, không phải server có dữ liệu thật.

1. Tạo user thử nghiệm: `sudo useradd -m labuser`, xác nhận user này **không** có quyền `sudo`.
2. Thêm `labuser` vào nhóm `docker`: `sudo usermod -aG docker labuser`.
3. Đăng nhập bằng `labuser` (hoặc `su - labuser`), xác nhận `docker ps` chạy được mà không cần `sudo`.
4. Với quyền `labuser`, chạy: `docker run -v /:/host -it alpine chroot /host cat /etc/shadow` — quan sát: `labuser` đọc được nội dung file mà chính user này (nếu không qua Docker) tuyệt đối không có quyền đọc trực tiếp.
5. Ghi lại bằng chứng: `id labuser` (không có root/sudo), nhưng kết quả bước 4 tương đương quyền root tuyệt đối trên host.
6. Dọn dẹp: gỡ `labuser` khỏi nhóm `docker` (`sudo gpasswd -d labuser docker`), xóa user thử nghiệm (`sudo userdel -r labuser`).

**Kết quả cần đạt**: báo cáo ngắn (lưu vào `labs/security-finding.md`) mô tả lại thực nghiệm này như một báo cáo đánh giá rủi ro thực tế bạn có thể trình bày trong công việc — đây là kỹ năng viết security finding thường gặp ở vị trí Sysadmin/DevOps cấp cao hơn.
