---
title: "07 - Interview"
module: 13
tags: [docker, sysops-infra, module-13, interview]
---

# 07 - Interview: Storage

## Junior

**1. Docker có bao nhiêu loại storage chính? Kể tên.**
3 loại: Bind Mount (mượn thư mục có sẵn trên host), Named Volume (Docker tự tạo và quản lý), tmpfs Mount (lưu trong RAM).

**2. Vì sao dữ liệu trong container bị mất khi `docker rm`?**
Nếu không dùng volume/bind mount, dữ liệu nằm trong writable layer (OverlayFS) của container — layer này chỉ tồn tại trong vòng đời container đó, bị xóa vĩnh viễn khi `docker rm`.

**3. Named volume và bind mount khác nhau cơ bản ở điểm nào?**
Bind mount là thư mục có sẵn trên host do người dùng chỉ định đường dẫn tường minh, Docker không quản lý. Named volume do Docker tạo và quản lý hoàn toàn (lưu tại `/var/lib/docker/volumes/...`), chỉ cần gọi bằng tên, portable hơn giữa các host.

**4. tmpfs mount có đặc điểm gì khác biệt so với hai loại còn lại?**
Dữ liệu lưu hoàn toàn trong RAM, biến mất ngay khi container dừng, không bao giờ chạm đĩa — tốc độ nhanh nhất nhưng không bền vững.

**5. Vì sao named volume được khuyến nghị cho dữ liệu production thay vì bind mount?**
Portable hơn (không phụ thuộc cấu trúc thư mục host cụ thể), ít rủi ro permission hơn, tách biệt vòng đời dữ liệu khỏi vòng đời container, và có quy trình backup/restore chuẩn hóa dễ áp dụng nhất quán trên nhiều host.

## Mid-level

**6. Giải thích cơ chế backup một named volume mà không cần cài thêm công cụ ngoài Docker.**
Chạy một container tạm (thường `alpine`), mount volume cần backup ở chế độ `:ro`, đồng thời bind mount thêm một thư mục host để nhận file kết quả, chạy `tar czf` bên trong để nén dữ liệu volume ra file `.tar.gz` lưu ở bind mount đó. Container tạm bị xóa sau khi chạy xong (`--rm`) nhưng file backup vẫn tồn tại độc lập trên host.

**7. Anonymous volume là gì, vì sao nó nguy hiểm?**
Khi Dockerfile khai báo `VOLUME` nhưng lúc `docker run` không chỉ định volume/bind mount tường minh cho đường dẫn đó, Docker tự tạo volume có tên là chuỗi hash ngẫu nhiên. Nguy hiểm vì: (1) `docker rm -v` xóa luôn anonymous volume mà không ai nhận ra đang xóa dữ liệu gì; (2) chạy lại container nhiều lần có thể tạo thêm nhiều anonymous volume mới, volume cũ mồ côi tích tụ chiếm dung lượng.

**8. Giải thích vì sao lỗi `permission denied` phổ biến với bind mount nhưng ít gặp hơn với named volume.**
Bind mount chia sẻ trực tiếp UID/GID số nguyên giữa host và container, không có tầng ánh xạ nào — nếu UID tiến trình container khác UID sở hữu thư mục host, kernel từ chối ghi ngay ở tầng filesystem. Named volume do Docker tự tạo mới (driver `local`) thường được khởi tạo với ownership phù hợp ngay từ đầu, và không phụ thuộc cấu trúc quyền có sẵn của một thư mục host cụ thể.

**9. `-v` và `--mount` khác nhau thế nào về mặt cú pháp và hành vi?**
`-v` (source:destination[:mode]) ngắn gọn nhưng có hành vi ngầm định: nếu source không phải đường dẫn tuyệt đối và chưa tồn tại như volume, Docker tự tạo named volume mới với tên đó — dễ gây nhầm lẫn nếu gõ sai. `--mount` yêu cầu khai báo tường minh `type=volume` hoặc `type=bind`, rõ ràng hơn, khuyến nghị dùng trong script tự động hóa để giảm rủi ro hành vi ngoài ý muốn.

**10. Vì sao nên mount volume nguồn ở chế độ `:ro` khi thực hiện backup?**
Đảm bảo container tạm thực hiện backup không thể vô tình sửa đổi dữ liệu gốc trong lúc đọc để nén — nguyên tắc an toàn cơ bản khi thao tác trên dữ liệu quan trọng, tránh trường hợp lỗi trong script backup (ví dụ nhầm lệnh) gây hư hại dữ liệu sản xuất thay vì chỉ đọc.

## Thực chiến

**11. Một sysadmin junior vô tình chạy `docker rm -f mydb` mà không kiểm tra trước container này có dùng volume hay không, dữ liệu database mất hoàn toàn không backup. Với vai trò senior, bạn thiết kế quy trình/kiểm soát nào để tình huống này không lặp lại trong tổ chức?**
Thiết kế nhiều lớp kiểm soát: (1) chuẩn hóa mọi container có dữ liệu quan trọng phải chạy qua template/script triển khai cố định (Ansible playbook hoặc Compose file) luôn kèm named volume tường minh, không cho phép `docker run` tay không kiểm soát trên production; (2) thiết lập lịch backup tự động định kỳ cho mọi named volume gắn với dữ liệu quan trọng (theo quy trình đã học ở Lab 6), không phụ thuộc con người nhớ backup thủ công; (3) giới hạn quyền `docker rm -f` trực tiếp trên node production cho tài khoản cá nhân, ưu tiên thao tác qua pipeline có review; (4) đào tạo/checklist bắt buộc trước khi thao tác xóa container: luôn `docker inspect --format '{{json .Mounts}}'` kiểm tra volume gắn kèm trước khi `rm`.

**12. Team dev báo ứng dụng chạy trong container ghi log liên tục vào một bind mount, nhưng sau vài ngày ổ đĩa host đầy dù `docker system df` báo dung lượng volume/image bình thường. Giải thích khả năng nguyên nhân và cách xác nhận.**
`docker system df` chỉ tính dung lượng image, container, **volume Docker quản lý**, và build cache — bind mount trỏ tới thư mục host **không được Docker tính vào con số này** vì về bản chất nó chỉ là một thư mục host bình thường, Docker không "sở hữu" nó. Xác nhận bằng `du -sh <đường-dẫn-bind-mount>` trực tiếp trên host, hoặc `df -h` theo từng phân vùng để tìm đúng nơi dung lượng bị chiếm. Đây là lý do quan trọng cần nhớ: giám sát dung lượng đĩa cho ứng dụng dùng bind mount phải giám sát riêng thư mục đó, không thể chỉ dựa vào `docker system df`.

**13. Bạn cần di chuyển (migrate) một database production đang chạy trên named volume từ server A sang server B, với thời gian downtime tối thiểu. Trình bày quy trình từng bước, chỉ dùng công cụ Docker thuần (không có shared storage giữa hai server).**
Bước 1: trên server A, thực hiện backup volume bằng container tạm như đã học (`tar czf`), trong khi database vẫn đang chạy nếu có thể chấp nhận backup "hot" (không nhất quán tuyệt đối) hoặc dừng ngắn database trước khi backup để đảm bảo tính nhất quán dữ liệu (tùy yêu cầu SLA). Bước 2: truyền file backup sang server B qua kênh an toàn (`scp`/`rsync`). Bước 3: trên server B, tạo named volume mới, restore file backup vào đó bằng container tạm (`tar xzf`). Bước 4: khởi động container database mới trên server B trỏ đúng volume vừa restore, xác nhận dữ liệu đầy đủ và ứng dụng kết nối được. Bước 5: chuyển traffic/DNS trỏ sang server B, chỉ dừng hẳn database ở server A sau khi đã xác nhận server B hoạt động ổn định — giữ server A như phương án rollback trong một khoảng thời gian an toàn trước khi dọn dẹp hẳn.

**14. Security team yêu cầu đảm bảo secret tạm thời (ví dụ private key dùng để giải mã dữ liệu lúc runtime) không bao giờ được ghi ra đĩa dưới bất kỳ hình thức nào, kể cả trong backup/snapshot filesystem host. Bạn thiết kế container xử lý secret này dùng loại storage nào, và vì sao các loại khác không đáp ứng được yêu cầu?**
Dùng tmpfs mount cho vị trí lưu secret tạm thời trong lúc xử lý — vì tmpfs hoàn toàn nằm trong RAM, không bao giờ được kernel ghi ra đĩa vật lý dưới bất kỳ hình thức nào (kể cả không vô tình bị chụp vào snapshot/backup filesystem host, vì nó không tồn tại trên filesystem thật). Named volume và bind mount đều loại trừ ngay từ đầu vì cả hai đều ghi dữ liệu thật ra đĩa host (dù ở vị trí Docker quản lý hay do người dùng chỉ định) — bất kỳ công cụ backup/snapshot filesystem nào chạy trên host đều có khả năng vô tình chụp lại nội dung đó, vi phạm trực tiếp yêu cầu bảo mật đã đặt ra.
