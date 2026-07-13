---
title: "06 - Troubleshooting"
module: 13
tags: [docker, sysops-infra, module-13, troubleshooting]
---

# 06 - Troubleshooting: Storage

## Sự cố 1: `permission denied` khi ứng dụng trong container ghi vào bind mount

**Chẩn đoán**:
```bash
docker run --rm <image> id                    # UID/GID tien trinh ben trong container
ls -ln <đường-dẫn-host>                        # UID/GID so huu thu muc tren host
```

**Nguyên nhân gốc**: bind mount chia sẻ trực tiếp UID/GID số nguyên giữa host và container — không có ánh xạ tự động. Nếu tiến trình bên trong container chạy với UID 1000 nhưng thư mục host thuộc sở hữu UID 1001 (hoặc root, mode không cho phép ghi), thao tác ghi bị kernel từ chối ngay ở tầng filesystem, không liên quan gì tới Docker.

**Khắc phục**: `chown` thư mục host khớp đúng UID container (`sudo chown -R 1000:1000 <thư-mục-host>`), hoặc chạy container với `--user $(id -u):$(id -g)` để ép tiến trình bên trong dùng đúng UID hiện tại của bạn trên host, hoặc (với Dockerfile do bạn kiểm soát) build image tạo user với UID cố định trùng khớp quy ước hạ tầng.

## Sự cố 2: Volume "biến mất" dữ liệu sau khi đổi image version

**Triệu chứng**: nâng cấp image database lên version mới, container mới chạy lên nhưng dữ liệu cũ không còn thấy.

**Nguyên nhân thường gặp**: image mới đổi đường dẫn dữ liệu mặc định bên trong (ví dụ từ `/var/lib/mysql` sang một đường dẫn khác trong version mới, hoặc ngược lại với các fork/biến thể image khác nhau), trong khi cấu hình `-v` cũ vẫn trỏ tới đường dẫn cũ — dữ liệu thật ra vẫn còn nguyên trong volume, chỉ là image mới không đọc đúng chỗ.

**Chẩn đoán**:
```bash
docker volume inspect <volume> --format '{{.Mountpoint}}'
sudo ls -la <mountpoint>   # xac nhan du lieu that su van con
docker inspect <container> --format '{{json .Mounts}}'
```

**Khắc phục**: kiểm tra changelog/tài liệu image mới về đường dẫn dữ liệu mặc định trước khi nâng cấp, đảm bảo `-v`/`--mount` trỏ đúng đường dẫn image mới thực sự sử dụng.

## Sự cố 3: Disk `/var/lib/docker` đầy vì anonymous volume tích tụ

**Chẩn đoán**:
```bash
docker system df -v
docker volume ls -f dangling=true      # liet ke volume mo coi (khong container nao dung)
```

**Nguyên nhân**: nhiều lần chạy container từ image có khai báo `VOLUME` trong Dockerfile mà không chỉ định tên volume/bind mount tường minh — mỗi lần tạo container mới, Docker âm thầm tạo thêm anonymous volume mới, volume cũ bị "mồ côi" (container tham chiếu đã bị xóa) nhưng dữ liệu vẫn còn, chiếm dung lượng vô thời hạn nếu không dọn.

**Khắc phục**: `docker volume prune` để dọn volume mồ côi (kiểm tra kỹ trước bằng `docker volume ls -f dangling=true` để chắc chắn không có volume nào quan trọng bị coi nhầm là mồ côi), và về lâu dài luôn chỉ định tên volume tường minh trong mọi lệnh `docker run` cho container có dữ liệu cần giữ.

## Sự cố 4: Backup volume ra file `.tar.gz` nhưng restore lại thiếu dữ liệu/permission sai

**Nguyên nhân thường gặp**: quên cờ `-C` khi `tar czf`, khiến đường dẫn lưu trong file tar chứa cả tiền tố thư mục tuyệt đối (`/source/...`) thay vì đường dẫn tương đối — khi giải nén (`tar xzf ... -C /target`), file bị giải nén sai cấu trúc thư mục (tạo thêm thư mục con `source/` lồng bên trong thay vì giải thẳng vào `/target`).

**Chẩn đoán**: `tar tzf <file-backup>.tar.gz | head` — kiểm tra đường dẫn bên trong archive có bắt đầu bằng `/` (tuyệt đối, dấu hiệu sai) hay là đường dẫn tương đối gọn (`./file1`, `./subdir/file2`, dấu hiệu đúng).

**Khắc phục**: luôn dùng đúng cú pháp `tar czf /backup/x.tar.gz -C /source .` (thay đổi thư mục làm việc bằng `-C` trước khi liệt kê `.` làm nội dung cần nén) như đã hướng dẫn ở [[04-Commands]] — không nén trực tiếp bằng đường dẫn tuyệt đối `tar czf x.tar.gz /source`.

## Sự cố 5: Container database khởi động rất chậm sau khi chuyển từ bind mount sang named volume, hoặc ngược lại

**Nguyên nhân cần điều tra**: nếu host dùng filesystem mạng (NFS) cho thư mục bind mount, hiệu năng I/O có thể kém hơn đáng kể so với named volume driver `local` (lưu trên đĩa cục bộ của host). Ngược lại, nếu named volume driver được cấu hình trỏ tới một remote storage backend (qua plugin), hiệu năng cũng phụ thuộc hoàn toàn vào backend đó, không còn là "tốc độ đĩa cục bộ" như named volume driver `local` mặc định.

**Chẩn đoán**: `docker volume inspect <volume>` xem `Driver` đang dùng gì; nếu là bind mount, kiểm tra `mount | grep <đường-dẫn>` xem thư mục đó có đang mount qua NFS/network filesystem hay không.

**Khắc phục**: với workload I/O nặng (database production), ưu tiên named volume driver `local` trên đĩa cục bộ (SSD nếu có), tránh đặt dữ liệu database quan trọng trên bind mount trỏ tới network filesystem trừ khi đã kiểm chứng kỹ hiệu năng và độ tin cậy của backend đó.

## Sự cố 6: `docker volume rm` báo lỗi "volume is in use" dù `docker ps` không thấy container nào đang chạy dùng nó

**Chẩn đoán**:
```bash
docker ps -a --filter volume=<tên-volume>
```

**Nguyên nhân**: container tham chiếu volume đó có thể đang ở trạng thái `exited`/`created` (không `running`) nhưng vẫn được Docker coi là "đang tham chiếu" volume — `docker ps` mặc định (không `-a`) chỉ hiện container running nên dễ bị bỏ sót.

**Khắc phục**: xác định đúng container (kể cả đã exited) bằng `docker ps -a --filter volume=`, xóa hẳn container đó (`docker rm`) trước khi `docker volume rm` mới thành công.
