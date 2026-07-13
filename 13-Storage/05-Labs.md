---
title: "05 - Labs"
module: 13
tags: [docker, sysops-infra, module-13, labs, hands-on]
---

# 05 - Labs: Storage

Môi trường: VM Linux đã cài Docker Engine 29.x. Lưu kết quả vào [[labs/README|labs/]].

## Lab 1 (Basic): Tái hiện sự cố mất dữ liệu do không dùng volume

1. Chạy `docker run -d --name mydb -e MYSQL_ROOT_PASSWORD=labpass mysql:8.4` (không có `-v` nào).
2. Kết nối vào MySQL, tạo một database và bảng chứa vài dòng dữ liệu mẫu.
3. `docker rm -f mydb`, chạy lại y hệt lệnh ở bước 1.
4. Kết nối lại, kiểm tra database vừa tạo — xác nhận dữ liệu đã mất hoàn toàn.
5. Viết ghi chú giải thích: dữ liệu đã nằm ở đâu trước khi mất (writable layer), vì sao `docker rm` xóa được nó.

## Lab 2 (Basic): Named Volume — dữ liệu sống sót qua vòng đời container

1. Lặp lại thí nghiệm Lab 1 nhưng lần này thêm `-v mydb-data:/var/lib/mysql`.
2. Tạo dữ liệu mẫu tương tự, `docker rm -f mydb`, chạy lại container mới với **cùng** `-v mydb-data:/var/lib/mysql`.
3. Xác nhận dữ liệu vẫn còn nguyên.
4. Kiểm tra vị trí vật lý thật của volume: `docker volume inspect mydb-data --format '{{.Mountpoint}}'`, dùng `sudo ls -la` xem trực tiếp file dữ liệu MySQL nằm ở đó.

## Lab 3 (Basic): Bind Mount — permission denied và cách khắc phục

1. Tạo thư mục trên host: `mkdir -p ~/lab13-bind && echo "test" > ~/lab13-bind/file.txt`.
2. Chạy container với image chạy bằng non-root user (ví dụ `node:20-alpine` chạy `node -e "require('fs').writeFileSync('/data/out.txt','ok')"`), bind mount `~/lab13-bind:/data`.
3. Quan sát lỗi `permission denied` nếu UID bên trong container khác UID sở hữu thư mục trên host.
4. Kiểm tra UID thực tế: `docker run --rm node:20-alpine id`, đối chiếu với `ls -ln ~/lab13-bind`.
5. Khắc phục bằng 1 trong 2 cách: (a) `chown` thư mục host theo đúng UID container, hoặc (b) chạy container với `--user $(id -u):$(id -g)` để khớp UID hiện tại của bạn trên host. Ghi lại cách nào phù hợp hơn cho tình huống dev cụ thể này.

## Lab 4 (Intermediate): tmpfs — xác nhận dữ liệu không chạm đĩa

1. Chạy `docker run -d --name cache-test --tmpfs /app/cache:size=50m alpine sleep 3600`.
2. `docker exec cache-test sh -c "dd if=/dev/zero of=/app/cache/testfile bs=1M count=10"`.
3. Xác nhận file tồn tại bên trong container: `docker exec cache-test ls -lh /app/cache`.
4. Từ host, thử tìm file này trên đĩa thật (`sudo find / -name testfile 2>/dev/null`) — xác nhận **không tìm thấy** vì dữ liệu chỉ nằm trong RAM.
5. `docker rm -f cache-test`, giải thích tại sao không cần "dọn dẹp" gì thêm ở tầng đĩa.

## Lab 5 (Intermediate): Backup và Restore hoàn chỉnh

1. Dùng lại volume `mydb-data` từ Lab 2 (đã có dữ liệu mẫu).
2. Thực hiện backup theo đúng quy trình ở [[04-Commands]] mục 5, xác nhận file `.tar.gz` được tạo ra và có dung lượng hợp lý.
3. Tạo volume mới `mydb-data-restored`, restore file backup vào đó theo mục 6.
4. Chạy một container MySQL **mới**, gắn `-v mydb-data-restored:/var/lib/mysql`, xác nhận dữ liệu mẫu xuất hiện đầy đủ và đúng — đây chính là bằng chứng quy trình backup/restore hoạt động thật, không chỉ "có vẻ đúng".
5. Đo thời gian thực hiện backup và restore, ghi lại cùng dung lượng dữ liệu gốc để có cơ sở ước lượng cho dữ liệu lớn hơn sau này.

## Lab 6 (Advanced/Mini Project): Quy trình backup định kỳ tự động cho database

**Mục tiêu**: xây dựng một script hoàn chỉnh có thể đưa vào cron thật, backup định kỳ một volume database, tự động dọn backup cũ.

**Yêu cầu**:
1. Viết script bash `backup-volume.sh` nhận tham số: tên volume, thư mục lưu backup, số lượng bản backup tối đa giữ lại.
2. Script phải: tạo tên file backup có timestamp, chạy container tạm để nén volume (giống Lab 5), sau đó tự động xóa các bản backup cũ vượt quá số lượng giữ lại đã cấu hình (ví dụ chỉ giữ 7 bản gần nhất).
3. Thêm bước kiểm tra: sau khi backup xong, tự động thử giải nén file `.tar.gz` vào một thư mục tạm (`tar tzf` để kiểm tra tính toàn vẹn archive, không cần restore thật vào volume) — phát hiện sớm nếu file backup bị hỏng.
4. Test script bằng cách chạy thủ công nhiều lần, xác nhận đúng số lượng file được giữ lại theo cấu hình.
5. Viết ghi chú (không phải câu hỏi phỏng vấn) mô tả cách bạn sẽ đưa script này vào `cron` thật trong môi trường production, và cách bạn sẽ giám sát nếu một lần backup thất bại (ví dụ script trả về exit code khác 0, tích hợp với hệ thống alerting).

**Kết quả cần đạt**: script `labs/backup-volume.sh` chạy được thật, kèm bằng chứng test (log output, danh sách file backup trước/sau khi dọn dẹp).
