---
title: "07 - Interview"
module: 15
tags: [docker, sysops-infra, module-15, interview]
---

# 07 - Interview: Multi-container Applications

## Junior

**1. Vai trò của Nginx reverse proxy trong một stack multi-container là gì?**
Là cổng vào duy nhất của hệ thống, nhận traffic từ client (thường qua HTTPS), xử lý TLS termination, cân bằng tải giữa nhiều instance backend, và che giấu toàn bộ backend/database khỏi mạng công cộng — chỉ Nginx publish port ra ngoài.

**2. Vì sao Redis và RabbitMQ không thể thay thế nhau dù đều là "trung gian"?**
Redis là kho lưu trữ key-value trong RAM, tối ưu cho đọc/ghi cực nhanh, dùng làm cache và session store — nằm trong luồng xử lý đồng bộ của request. RabbitMQ là message broker, triển khai mô hình hàng đợi để tách một tác vụ chậm ra khỏi luồng phản hồi chính, xử lý bất đồng bộ bởi worker riêng biệt.

**3. Vì sao database bắt buộc phải dùng named volume trong stack này, không có ngoại lệ?**
Vì writable layer của container không phải nơi lưu dữ liệu lâu dài (đã học Module 08/13) — mất dữ liệu database gần như luôn đồng nghĩa mất dữ liệu khách hàng/giao dịch, không thể phục hồi nếu không có backup hoặc volume bền vững.

**4. Trong mô hình network phân vùng của stack, thành phần nào được phép nằm trong cả hai network `frontend` và `backend`?**
Chỉ `backend` (service ứng dụng) — đóng vai trò cầu nối duy nhất giữa vùng tiếp xúc client (`frontend`, có Nginx) và vùng dữ liệu nội bộ (`backend`, có database/Redis/RabbitMQ).

**5. Producer và Consumer trong mô hình message queue là gì?**
Producer là thành phần đẩy (publish) message vào queue — trong stack này là backend. Consumer là thành phần lấy (consume) message ra xử lý — trong stack này là worker.

## Mid-level

**6. Giải thích vì sao backend cần được thiết kế "stateless", liên hệ trực tiếp tới khả năng scale.**
Nếu backend tự lưu trạng thái (session, cache tạm) trong bộ nhớ của chính nó, việc scale thành nhiều instance sẽ gây lỗi: request của cùng một người dùng có thể rơi vào các instance khác nhau (do load balancing round-robin của Nginx) không có state đó — gây hiện tượng "đăng xuất ngẫu nhiên" hoặc mất dữ liệu tạm. Thiết kế stateless (mọi state cần thiết nằm ở database/Redis dùng chung) cho phép thêm/bớt instance backend tự do mà không ảnh hưởng tính đúng đắn của hệ thống.

**7. Vì sao RabbitMQ management UI nên gắn `profiles: ["debug"]` thay vì luôn bật?**
Giảm bề mặt tấn công — management UI là một giao diện quản trị có quyền truy cập sâu vào queue, không cần thiết phải luôn sẵn sàng truy cập được trên production. Gắn profile cho phép chỉ bật khi cần debug tường minh (`--profile debug`), đúng nguyên tắc least privilege đã áp dụng xuyên suốt từ Module 08 (user non-root) tới Module 14 (profiles).

**8. Giải thích sự khác biệt giữa nén trực tiếp thư mục dữ liệu database bằng `tar` và dùng `pg_dump`/`mysqldump` khi backup.**
`tar` nén trực tiếp file vật lý trên đĩa — nếu database đang chạy và ghi dữ liệu liên tục, có rủi ro chụp phải trạng thái file không nhất quán (đặc biệt write-ahead log). `pg_dump`/`mysqldump` là công cụ dump logic của chính database, đảm bảo tính nhất quán tại một thời điểm snapshot dù database vẫn đang phục vụ traffic — đây là phương pháp đúng cho backup database đang chạy live.

**9. Khi scale worker lên nhiều instance (`docker compose up -d --scale worker=3`), RabbitMQ phân phối message giữa các worker theo cơ chế nào?**
RabbitMQ phân phối message theo cơ chế round-robin mặc định giữa các consumer đang kết nối vào cùng một queue — mỗi message chỉ được một consumer xử lý (không phải mọi worker đều nhận cùng một message), cho phép tăng throughput xử lý tuyến tính theo số lượng worker (trong giới hạn tài nguyên hệ thống).

**10. Vì sao cần kiểm tra healthcheck của backend thực sự kiểm tra đúng cổng HTTP mà Nginx gọi tới, không chỉ kiểm tra kết nối database?**
Healthcheck chỉ kiểm tra một phần của hệ thống (ví dụ chỉ kết nối database) có thể báo `healthy` trong khi cổng HTTP thực sự chưa sẵn sàng lắng nghe (do thứ tự khởi tạo code sai hoặc lỗi bind port) — Nginx dựa vào việc backend "healthy" để forward traffic, nếu healthcheck không phản ánh đúng khả năng phục vụ HTTP thực tế, kết quả là lỗi `502 Bad Gateway` dù mọi thứ "trông có vẻ" ổn theo `docker compose ps`.

## Thực chiến

**11. Sau khi triển khai stack lên production, team nhận thấy mỗi lần Redis restart (do bảo trì node), toàn bộ hệ thống chậm hẳn trong vài phút trước khi ổn định lại. Đây có phải sự cố cần sửa không, và nếu có, hướng khắc phục là gì?**
Đây là hiện tượng "cache khởi động lạnh" (cold cache) — bình thường và có thể chấp nhận được ở mức độ nhất định, nhưng nếu ảnh hưởng đáng kể tới trải nghiệm người dùng, cần cân nhắc giải pháp: (1) bật Redis persistence (RDB/AOF) để cache không mất hoàn toàn khi restart, giảm mức độ "lạnh"; (2) triển khai Redis ở chế độ có replica/sentinel (nâng cao hơn phạm vi module này) để bảo trì một node không làm mất toàn bộ cache; (3) thiết kế cơ chế "cache warming" chủ động sau khi Redis khởi động lại, chủ động nạp trước các dữ liệu hay truy cập nhất thay vì chờ traffic thật kích hoạt cache miss dần dần.

**12. Một buổi tối, hệ thống nhận traffic tăng đột biến (sale campaign), `worker` không xử lý kịp khiến hàng nghìn email xác nhận đơn hàng bị trễ nhiều giờ. Đội vận hành cần phản ứng ngay trong sự cố, và cần đề xuất cải tiến kiến trúc dài hạn sau đó. Trình bày cả hai.**
Phản ứng ngay: scale worker tạm thời (`docker compose up -d --scale worker=N` với N đủ lớn để xử lý backlog nhanh hơn), theo dõi RabbitMQ management UI để xác nhận queue đang giảm dần, không tăng quá mức gây quá tải ngược lên database (worker thường cũng ghi kết quả xử lý vào database). Cải tiến dài hạn: thiết lập autoscaling worker dựa trên độ dài queue (vượt phạm vi Compose thuần túy, đây chính là bài toán Kubernetes Horizontal Pod Autoscaler dựa trên custom metric sẽ giải quyết tốt hơn ở khóa Kubernetes tiếp theo), tách riêng queue ưu tiên cao (ví dụ email xác nhận đơn hàng) khỏi queue ưu tiên thấp (ví dụ email marketing) để sự cố tồn đọng ở một loại tác vụ không làm trễ tác vụ quan trọng hơn.

**13. Security team phát hiện container `db` có thể bị truy cập trực tiếp từ container `nginx` dù thiết kế ban đầu không cho phép. Kiểm tra cho thấy compose.yaml đúng như tài liệu (db chỉ ở network `backend`). Nguyên nhân khả dĩ nào cần điều tra tiếp?**
Khả năng cao nhất: có một service khác (không phải `nginx` gốc trong file chính) đã được `docker network connect` thủ công vào network `backend` sau khi stack đã chạy (ví dụ ai đó debug và quên gỡ kết nối), hoặc có một `compose.override.yaml` (Compose tự động merge file này nếu tồn tại cùng thư mục) âm thầm thêm `nginx` vào network `backend` mà không ai để ý khi chỉ đọc file `compose.yaml` chính. Điều tra bằng `docker network inspect backend` để liệt kê chính xác container nào đang thực sự kết nối, đối chiếu với danh sách mong đợi từ `compose.yaml`; kiểm tra toàn bộ thư mục project có file `compose.override.yaml`/`compose.*.yaml` nào khác đang được merge ngầm hay không.

**14. Đội kiến trúc đề xuất tách stack này thành nhiều Compose project riêng biệt (database + Redis + RabbitMQ là một project "hạ tầng dùng chung", nginx + backend + worker là project "ứng dụng") để nhiều team ứng dụng khác nhau cùng dùng chung một cụm hạ tầng. Thiết kế network nào cho phép hai project Compose độc lập giao tiếp được với nhau?**
Dùng **external network**: tạo một network trước bằng `docker network create shared-infra-net` (hoặc để project "hạ tầng" tạo và khai báo `external: false` cho chính nó), sau đó project "ứng dụng" khai báo network đó với `external: true` trong `compose.yaml` của mình để tham chiếu tới network đã tồn tại thay vì tự tạo mới. Cả hai project độc lập về vòng đời (`up`/`down` riêng, project name riêng, container riêng) nhưng cùng gắn vào một network vật lý chung, cho phép service ở project này resolve tên service ở project kia qua embedded DNS như đã học ở Module 12/14 — đây chính xác là mô hình "shared infrastructure network" phổ biến trong tổ chức có nhiều team cùng dùng chung một cụm database/message queue.
