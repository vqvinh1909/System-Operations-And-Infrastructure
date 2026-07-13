---
title: "05 - Labs"
module: 15
tags: [docker, sysops-infra, module-15, labs, hands-on]
---

# 05 - Labs: Multi-container Applications

Môi trường: VM Linux đã cài Docker Engine 29.x kèm Compose plugin nhánh 5.x, đủ RAM để chạy đồng thời 6 container (khuyến nghị tối thiểu 4GB RAM cho VM lab). Lưu toàn bộ code vào [[labs/README|labs/]].

## Lab 1 (Basic): Dựng từng thành phần độc lập trước khi ghép

1. Chạy riêng lẻ từng thành phần bằng `docker run` (chưa dùng Compose): `postgres:16-alpine`, `redis:7-alpine`, `rabbitmq:4-management-alpine` — xác nhận từng cái khởi động thành công độc lập, kiểm tra bằng `docker logs`.
2. Với mỗi thành phần, tìm đúng lệnh healthcheck phù hợp (`pg_isready`, `redis-cli ping`, `rabbitmq-diagnostics ping`) — thử chạy tay các lệnh này qua `docker exec` để xác nhận chúng thực sự phản ánh đúng trạng thái sẵn sàng.
3. Dọn dẹp toàn bộ container thử nghiệm.

**Kết quả cần đạt**: hiểu rõ từng thành phần hoạt động độc lập ra sao trước khi ghép chúng — tránh tình trạng debug một stack phức tạp mà không biết lỗi nằm ở thành phần nào.

## Lab 2 (Basic): Viết một backend tối giản kết nối cả 3 dependency

1. Viết một ứng dụng backend tối giản (ngôn ngữ tùy chọn — Python/Node.js) có 3 chức năng: (a) endpoint `/health` trả 200 khi kết nối được cả database, Redis, RabbitMQ; (b) endpoint ghi/đọc một giá trị vào Postgres; (c) endpoint đẩy một message thử nghiệm vào RabbitMQ.
2. Viết Dockerfile cho backend theo đúng best practice Module 10 (multi-stage nếu cần, non-root user, healthcheck).
3. Test backend chạy độc lập bằng `docker run`, trỏ tới 3 dependency đã chạy riêng ở Lab 1 (dùng `--network` chung để chúng thấy nhau).

## Lab 3 (Intermediate): Ghép toàn bộ bằng compose.yaml, network phân vùng

1. Viết `compose.yaml` đầy đủ theo mẫu ở [[04-Commands]] mục 1, dùng backend đã viết ở Lab 2.
2. Chạy `docker compose up -d`, quan sát `docker compose ps` cho tới khi mọi service chuyển `healthy`.
3. Kiểm chứng phân vùng network: `docker compose exec nginx ping -c 2 db` phải thất bại (không cùng network); `docker compose exec backend ping -c 2 db` phải thành công.
4. Gọi thử `curl localhost:80/health` từ host — xác nhận response 200, chứng minh toàn bộ chuỗi Nginx → Backend → 3 dependency hoạt động.

## Lab 4 (Intermediate): Tái hiện race condition lúc khởi động, rồi khắc phục

1. Tạm thời xóa `healthcheck` của `db` và đổi `depends_on` của `backend` về short syntax.
2. `docker compose down -v` rồi `docker compose up -d` lại nhiều lần liên tiếp, quan sát log `backend` — cố gắng bắt được ít nhất một lần backend crash vì kết nối database quá sớm.
3. Khôi phục lại `healthcheck` + `condition: service_healthy` đúng chuẩn, lặp lại thử nghiệm 5-10 lần liên tiếp — xác nhận không còn crash loop nào xảy ra.
4. Ghi lại log bằng chứng (có/không có healthcheck) vào `labs/race-condition-evidence.md`.

## Lab 5 (Intermediate): Message queue thật sự hoạt động — worker tiêu thụ message

1. Viết một service `worker` tối giản (Dockerfile riêng hoặc image chung với backend, entrypoint khác) liên tục lắng nghe queue RabbitMQ, in ra log mỗi khi nhận message.
2. Thêm `worker` vào compose.yaml, gắn đúng network `backend` (không cần `frontend`).
3. Gọi endpoint đẩy message ở backend (từ Lab 2) nhiều lần liên tiếp, quan sát log `worker` xử lý từng message.
4. Dừng `worker` (`docker compose stop worker`), tiếp tục gọi endpoint đẩy message vài lần, sau đó khởi động lại `worker` — xác nhận các message đã "tồn đọng" trong lúc worker dừng vẫn được xử lý đầy đủ khi worker sống lại (minh chứng cho tính chịu lỗi của message queue đã học ở [[02-Theory]] mục 5.2).

## Lab 6 (Advanced/Mini Project): Stack hoàn chỉnh production-grade + quy trình vận hành đầy đủ

**Mục tiêu**: đây là bài tập tổng hợp cuối cùng của Part II, mô phỏng đầy đủ một ca vận hành thật.

**Yêu cầu**:
1. Hoàn thiện `compose.yaml` đạt chuẩn TẤT CẢ yêu cầu đã liệt kê ở [[04-Commands]] (secrets file-based, healthcheck đầy đủ, network phân vùng, profiles cho RabbitMQ management UI, `.env` cho giá trị không nhạy cảm).
2. Thực hiện đầy đủ vòng đời vận hành, ghi lại từng bước và bằng chứng:
   - Dựng stack từ đầu (`docker compose up -d`), xác nhận mọi service `healthy`.
   - Backup dữ liệu database bằng container tạm (theo Module 13), xác nhận file backup hợp lệ.
   - Thực hiện rolling update cho `backend` (sửa code nhỏ, `docker compose up -d --build backend`), xác nhận `db`/`redis`/`rabbitmq` không bị restart theo.
   - Mô phỏng sự cố: dừng đột ngột `redis` (`docker compose kill redis`), quan sát hành vi backend (nên vẫn phục vụ được, chỉ chậm hơn do cache miss liên tục — nếu backend crash hoàn toàn khi Redis chết, đây là lỗi thiết kế cần ghi nhận và sửa lại logic fallback).
   - Khôi phục `redis` (`docker compose up -d redis`), xác nhận hệ thống trở lại bình thường.
   - Dừng an toàn toàn bộ stack (`docker compose down`, giữ volume).
3. Viết một tài liệu vận hành ngắn (runbook, không phải câu hỏi phỏng vấn) mô tả: cách dựng stack, cách kiểm tra sức khỏe, cách backup, cách rolling update, cách xử lý khi một dependency (Redis/RabbitMQ) tạm thời không khả dụng.

**Sản phẩm nộp**: toàn bộ source code, `compose.yaml`, `.env.example`, `secrets/` (chỉ file mẫu), và runbook vận hành, lưu vào [[labs/README|labs/]].
