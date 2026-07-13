---
title: "Labs - Module 13"
module: 13
tags: [docker, sysops-infra, module-13, labs]
---

# Labs - Module 13: Storage

Thư mục lưu kết quả thực hành từ [[../05-Labs|05-Labs.md]].

## Môi trường

- VM Linux đã cài Docker Engine 29.x.
- Đủ dung lượng đĩa trống để test backup/restore (Lab 5, Lab 6) và tạo dữ liệu mẫu MySQL.

## File dự kiến sinh ra trong thư mục này

| File/Thư mục | Sinh ra ở Lab | Nội dung |
|---|---|---|
| `data-loss-evidence.md` | Lab 1 | Bằng chứng dữ liệu mất khi không dùng volume |
| `named-volume-evidence.md` | Lab 2 | Bằng chứng dữ liệu sống sót qua vòng đời container |
| `permission-fix-notes.md` | Lab 3 | Ghi chú UID/GID, cách khắc phục permission denied |
| `tmpfs-evidence.md` | Lab 4 | Bằng chứng file tmpfs không tồn tại trên đĩa thật |
| `backup/` | Lab 5, Lab 6 | File `.tar.gz` backup thực tế, bằng chứng restore thành công |
| `backup-volume.sh` | Lab 6 (mini project) | Script backup định kỳ hoàn chỉnh, có dọn backup cũ và kiểm tra toàn vẹn |

## Lưu ý an toàn

- Lab 1 cố tình tái hiện mất dữ liệu — chỉ dùng dữ liệu mẫu tự tạo, không dùng dữ liệu thật quan trọng.
- Luôn test restore thật (không chỉ backup) trước khi coi quy trình backup là hoàn chỉnh — đúng nguyên tắc đã nhấn mạnh ở [[../02-Theory|02-Theory.md]] mục 5.2.
