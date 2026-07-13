---
title: "03 - Architecture"
module: 13
tags: [docker, sysops-infra, module-13, architecture, diagram]
---

# 03 - Architecture: So sánh Storage, Luồng Backup/Restore

Toàn bộ sơ đồ đã được đo bằng script Python (so sánh `len()` border trên/dưới và mọi dòng nội dung trong từng khung) trước khi đưa vào file.

## 1. So sánh 3 loại storage

```text
+---------------------------------------------------------------+
|                   3 LOAI STORAGE CUA DOCKER                   |
+---------------------------------------------------------------+
|  Bind Mount   - muon thu muc co san tren host, native speed   |
+---------------------------------------------------------------+
| Named Volume - Docker tao va quan ly, tot nhat cho production |
+---------------------------------------------------------------+
|     tmpfs Mount  - luu trong RAM, mat khi container dung      |
+---------------------------------------------------------------+
```

Ba lựa chọn tương ứng ba mức độ kiểm soát khác nhau: bind mount người dùng tự quản lý hoàn toàn đường dẫn nguồn (linh hoạt nhưng kém portable), named volume giao phó cho Docker quản lý (portable, best practice production), tmpfs đánh đổi tính bền vững lấy tốc độ tuyệt đối và không bao giờ chạm đĩa.

## 2. Luồng backup volume qua container tạm

```text
+-----------------------------------------------------------+
| Volume mydata (du lieu goc, mount -v mydata:/source:ro)   |
+-----------------------------------------------------------+
                              |
                              v
+-----------------------------------------------------------+
|         Container tam alpine chay lenh tar czf            |
+-----------------------------------------------------------+
                              |
                              v
+-----------------------------------------------------------+
|      Bind mount host: (pwd)/backup nhan file ket qua      |
+-----------------------------------------------------------+
                              |
                              v
+-----------------------------------------------------------+
|         File mydata-20260713.tar.gz nam tren host         |
+-----------------------------------------------------------+
```

**Đọc sơ đồ**: container tạm đóng vai trò "cầu nối" duy nhất giữa hai thế giới độc lập — named volume (nơi Docker quản lý dữ liệu gốc) và bind mount host (nơi lưu file backup lâu dài, độc lập hoàn toàn với vòng đời của bất kỳ container hay volume nào). Mount volume nguồn ở chế độ `:ro` (read-only) là nguyên tắc an toàn bắt buộc — đảm bảo thao tác backup không thể vô tình sửa đổi dữ liệu gốc. Container tạm bị xóa ngay sau khi chạy xong (`--rm`), nhưng file kết quả vẫn tồn tại trên host vì nó nằm ở bind mount, không nằm trong writable layer của container tạm đó.

Quy trình restore đi ngược chiều: file `.tar.gz` trên host (bind mount) được giải nén vào một named volume mới (thường trống, mới `docker volume create`) thông qua cùng cơ chế container tạm — chỉ đảo vai trò nguồn/đích và đổi `tar czf` (nén) thành `tar xzf` (giải nén).

Tiếp theo: [[04-Commands]] để xem cú pháp lệnh cụ thể áp dụng toàn bộ khái niệm ở [[02-Theory]] và sơ đồ trên.
