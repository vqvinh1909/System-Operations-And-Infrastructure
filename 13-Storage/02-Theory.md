---
title: "02 - Theory"
module: 13
tags: [docker, sysops-infra, module-13, theory, volume, bind-mount, tmpfs]
---

# 02 - Theory: Bind Mount, Volume, tmpfs

## 1. Nhắc lại vấn đề gốc: writable layer không dùng để lưu dữ liệu lâu dài

Từ Module 08: container filesystem là OverlayFS, gồm nhiều layer chỉ đọc của image (`lowerdir`) cộng một layer ghi duy nhất (`upperdir` — writable layer) tồn tại đúng bằng vòng đời container. Cả ba cơ chế storage của Docker đều nhằm một mục đích: **đưa dữ liệu ra khỏi writable layer**, gắn thẳng vào một vị trí bền vững hơn (host filesystem thật, hoặc RAM), thông qua **mount namespace** (đã học Module 08) — kernel gắn thêm một mount point vào cây filesystem của container, đè lên đường dẫn chỉ định, độc lập với OverlayFS bên dưới.

## 2. Bind Mount

### 2.1 Cơ chế

Bind mount lấy một thư mục/file **có sẵn trên host** (theo đường dẫn tuyệt đối), gắn thẳng (mount) vào một đường dẫn bên trong container. Không có lớp trừu tượng nào ở giữa — container đọc/ghi thẳng vào đúng filesystem thật của host tại đường dẫn đó.

```bash
docker run -v /home/vinh/app-code:/app myapp:latest
```

### 2.2 Đặc điểm

- **Do người dùng quản lý hoàn toàn đường dẫn nguồn** — Docker không kiểm soát, không biết trước thư mục đó tồn tại hay không (nếu chưa tồn tại, Docker tự tạo thư mục trống — hành vi này từng gây bug khó hiểu khi gõ nhầm đường dẫn, dẫn tới container chạy với thư mục rỗng thay vì báo lỗi).
- **Phụ thuộc cấu trúc thư mục của host cụ thể** — kém portable, vì đường dẫn `/home/vinh/app-code` chỉ có ý nghĩa trên đúng máy đó.
- **Chia sẻ trực tiếp UID/GID** — file tạo ra bên trong container mang đúng UID/GID của tiến trình bên trong container đó, nhìn từ host là một UID/GID số nguyên bình thường. Đây là nguồn gốc phổ biến nhất của lỗi `permission denied` (chi tiết ở [[06-Troubleshooting]]).
- **Hiệu năng gần như native** — không qua OverlayFS, không copy-on-write, đọc/ghi trực tiếp filesystem host.

### 2.3 Khi nào dùng

- **Môi trường dev**: mount source code từ máy dev vào container để code thay đổi tức thì phản ánh vào container đang chạy (live reload) — không cần rebuild image mỗi lần sửa code.
- **Đưa file cấu hình cụ thể của host vào container**: ví dụ mount `/etc/localtime` (đồng bộ múi giờ), hoặc file cấu hình do công cụ quản lý cấu hình (Ansible) sinh ra sẵn trên host.
- **Không khuyến nghị cho dữ liệu production quan trọng** vì thiếu tính portable và dễ lỗi permission khi di chuyển giữa các host khác nhau.

## 3. Named Volume

### 3.1 Cơ chế

Named volume là một vùng lưu trữ **do chính Docker tạo ra và quản lý hoàn toàn**, nằm dưới `/var/lib/docker/volumes/<tên-volume>/_data` trên Linux host theo mặc định (driver `local`). Người dùng không cần biết/quan tâm đường dẫn vật lý thật — chỉ cần gọi tên volume.

```bash
docker volume create mydata
docker run -v mydata:/var/lib/mysql mysql:8.4
```

Nếu volume `mydata` chưa tồn tại, `docker run` **tự động tạo mới** khi thấy tên volume trong `-v` — khác biệt quan trọng với bind mount (Docker không tự "phát minh" đường dẫn host).

### 3.2 Đặc điểm

- **Docker quản lý toàn bộ vòng đời**: tạo, liệt kê, xóa qua `docker volume`, độc lập với vòng đời bất kỳ container cụ thể nào.
- **Portable hơn bind mount**: không phụ thuộc cấu trúc thư mục cụ thể của host — chỉ cần tên volume, hoạt động nhất quán trên mọi host có Docker.
- **Hỗ trợ volume driver plugin**: ngoài driver `local` mặc định, có thể dùng plugin để lưu volume trên NFS, cloud storage (bổ sung sau này khi cần multi-host, vượt phạm vi module này).
- **Được khởi tạo dữ liệu từ image nếu thư mục đích đã có sẵn nội dung**: nếu image có sẵn dữ liệu tại đường dẫn mount (ví dụ image MySQL đã có sẵn file cấu hình mặc định tại `/var/lib/mysql`), lần đầu tiên volume trống được mount vào, Docker copy nội dung có sẵn đó từ image vào volume — hành vi hữu ích, nhưng cũng là nguồn gây bối rối nếu không biết trước.

### 3.3 Vì sao named volume là best practice cho dữ liệu production

- **Tách biệt vòng đời dữ liệu khỏi vòng đời container**: `docker rm` container không đụng tới volume — đúng tình huống sysadmin junior ở [[01-Introduction]] lẽ ra tránh được nếu dùng named volume thay vì để dữ liệu mặc định rơi vào writable layer.
- **Hiệu năng tốt hơn writable layer**: named volume (driver `local`) không đi qua OverlayFS copy-on-write, đọc/ghi trực tiếp filesystem host tại vị trí Docker quản lý — quan trọng với workload I/O nặng như database.
- **Dễ backup/restore bằng container tạm** (chi tiết mục 5) — quy trình chuẩn hóa, không phụ thuộc biết trước đường dẫn vật lý cụ thể trên từng host.
- **Không lộ cấu trúc thư mục host ra ứng dụng** — giảm rủi ro bảo mật/vận hành khi di chuyển giữa các môi trường khác nhau (dev/staging/production có thể có cấu trúc thư mục host hoàn toàn khác nhau, nhưng tên volume logic giữ nguyên).

## 4. tmpfs Mount

### 4.1 Cơ chế

tmpfs mount tạo một filesystem tạm **hoàn toàn nằm trong RAM** của host, gắn vào container — không bao giờ chạm tới đĩa vật lý.

```bash
docker run --tmpfs /app/cache:size=100m,mode=1777 myapp:latest
```

### 4.2 Đặc điểm

- **Biến mất hoàn toàn khi container dừng** — không có writable layer nào được ghi ra đĩa, không có volume nào tồn tại sau đó để backup.
- **Tốc độ đọc/ghi cực nhanh** (tốc độ RAM) — không có I/O đĩa nào xảy ra.
- **Chỉ dùng được trên Linux container** (không hỗ trợ trên Windows container).
- **Tăng rủi ro nếu dùng sai mục đích**: dữ liệu nhạy cảm (secret, session key tạm thời) đặt ở tmpfs sẽ không bao giờ vô tình bị ghi ra đĩa (không lộ khi đọc raw disk/backup filesystem host) — đây thực chất là một **lợi ích bảo mật** khi dùng đúng mục đích.

### 4.3 Khi nào dùng

- Cache tạm thời không cần tồn tại qua lần restart container (ví dụ cache biên dịch, cache session xử lý ngắn hạn).
- Lưu trữ secret/key tạm thời trong lúc xử lý (tránh ghi ra đĩa, giảm rủi ro rò rỉ qua backup/snapshot filesystem host vô tình chứa dữ liệu nhạy cảm).
- Thư mục scratch (làm việc tạm) của tiến trình xử lý dữ liệu lớn cần I/O cực nhanh và không cần giữ lại kết quả trung gian.

## 5. Backup và Restore Volume — kỹ năng bắt buộc

### 5.1 Nguyên lý: dùng container tạm

Docker không có lệnh `docker volume backup` sẵn có. Kỹ thuật chuẩn (không cần cài thêm công cụ ngoài Docker) là: chạy một **container tạm** (thường dùng image tối giản như `alpine`), mount volume cần backup vào container đó, đồng thời mount thêm một bind mount tới thư mục host để lưu file backup, rồi chạy lệnh `tar` bên trong để nén dữ liệu.

```bash
# Backup: nen volume "mydata" thanh file tar.gz luu ra host
docker run --rm \
  -v mydata:/source:ro \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/mydata-$(date +%Y%m%d).tar.gz -C /source .
```

**Giải thích từng phần**: `-v mydata:/source:ro` mount volume cần backup ở chế độ chỉ đọc (đảm bảo không vô tình sửa dữ liệu gốc trong lúc backup — nguyên tắc an toàn cơ bản). `-v $(pwd)/backup:/backup` là bind mount thư mục host để nhận file kết quả (vì container tạm sẽ bị xóa ngay sau khi chạy xong, `--rm`, nên kết quả phải nằm ở nơi tồn tại độc lập với container đó). `tar czf ... -C /source .` nén toàn bộ nội dung volume, `-C` đổi thư mục làm việc trước khi nén để đường dẫn trong file tar gọn (không chứa tiền tố `/source/`).

### 5.2 Restore

```bash
# Tao volume moi (hoac dung volume rong da co san)
docker volume create mydata-restored

# Giai nen file backup vao volume moi
docker run --rm \
  -v mydata-restored:/target \
  -v $(pwd)/backup:/backup \
  alpine tar xzf /backup/mydata-20260713.tar.gz -C /target
```

**Nguyên tắc thực chiến**: luôn test quy trình restore trên một volume/môi trường thử nghiệm **trước khi cần dùng thật trong sự cố** — một quy trình backup chưa từng được test restore không đáng tin cậy, đây là nguyên tắc chung của mọi hệ thống backup, không riêng gì Docker.

### 5.3 Vì sao dùng `alpine` (hoặc image tối giản tương tự) cho container tạm

Container tạm chỉ cần công cụ `tar` (có sẵn trong hầu hết base image, kể cả `alpine` ~7MB) — không cần chạy ứng dụng thật, không cần image nặng. Việc này giữ quy trình backup/restore nhẹ, nhanh, và không phụ thuộc vào bất kỳ image ứng dụng cụ thể nào.

## 6. Anonymous Volume — cái bẫy hay gây mất dữ liệu

Nếu Dockerfile có khai báo `VOLUME /data` (đã học ở Module 10) nhưng lúc `docker run` **không** chỉ định volume/bind mount nào cho đường dẫn đó, Docker tự tạo một **anonymous volume** — một volume có tên là chuỗi hash ngẫu nhiên, không có tên gợi nhớ.

**Vấn đề thực tế**: `docker rm -v` (cờ `-v` khi xóa container) sẽ xóa luôn anonymous volume gắn với container đó — nhưng vì tên là hash ngẫu nhiên, không ai nhớ hay biết volume nào chứa dữ liệu gì để tránh xóa nhầm. Ngoài ra, mỗi lần `docker run` lại (không dùng `docker start` container cũ) mà không chỉ định tên volume tường minh, Docker có thể tạo **thêm một anonymous volume mới** — dữ liệu cũ vẫn còn trong volume ẩn cũ (không mất ngay, nhưng "biến mất" khỏi tầm nhìn của ứng dụng vì container mới không trỏ tới volume cũ), rác anonymous volume tích tụ dần theo thời gian, chiếm dung lượng đĩa mà không ai để ý.

**Khuyến nghị thực chiến**: luôn chỉ định tường minh named volume (`-v mydata:/var/lib/mysql`), không bao giờ để Docker tự tạo anonymous volume trong môi trường production.

## 7. Bảng tổng hợp so sánh 3 loại storage

| Tiêu chí | Bind Mount | Named Volume | tmpfs |
|---|---|---|---|
| Vị trí lưu trữ | Đường dẫn host do người dùng chỉ định | Docker quản lý (`/var/lib/docker/volumes/...`) | RAM |
| Docker quản lý vòng đời | Không | Có | Có (nhưng không tồn tại lâu dài) |
| Tồn tại sau khi container bị xóa | Có (là thư mục host, độc lập hoàn toàn) | Có (trừ khi chủ động `docker volume rm`) | Không — mất ngay khi container dừng |
| Hiệu năng | Native (không qua OverlayFS) | Tốt (driver `local` không qua OverlayFS) | Rất nhanh (RAM) |
| Portable giữa host khác nhau | Kém (phụ thuộc cấu trúc thư mục host) | Tốt (chỉ cần tên volume) | Không áp dụng (không persist) |
| Rủi ro permission | Cao (lệch UID/GID host-container) | Thấp hơn (Docker tự quản lý ownership khi tạo mới) | Không áp dụng |
| Use case điển hình | Mount code dev, file cấu hình cụ thể của host | Dữ liệu production: database, file upload | Cache tạm, secret tạm thời |

Xem sơ đồ trực quan so sánh và luồng backup/restore ở [[03-Architecture]].
