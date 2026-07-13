---
title: "07 - Interview"
module: 14
tags: [docker, sysops-infra, module-14, interview]
---

# 07 - Interview: Docker Compose

## Junior

**1. Docker Compose dùng để làm gì?**
Khai báo và quản lý nhiều container liên quan tới nhau (services, network, volume) trong một file YAML duy nhất, dựng toàn bộ hệ thống bằng một lệnh `docker compose up`, thay vì gõ nhiều lệnh `docker run` thủ công.

**2. Vì sao không cần khóa `version:` trong compose.yaml hiện tại?**
Từ khi cộng đồng hợp nhất thành Compose Specification, Compose CLI luôn dùng schema mới nhất để validate file, bỏ qua hoàn toàn khóa `version:` — best practice hiện tại là bỏ hẳn khóa này, đặt tên file `compose.yaml`.

**3. `depends_on` mặc định (short syntax) đảm bảo điều gì, không đảm bảo điều gì?**
Chỉ đảm bảo thứ tự container được **start** trước, không đảm bảo tiến trình bên trong dependency đã thực sự **sẵn sàng phục vụ** (ví dụ database đã accept connection). Muốn đảm bảo điều đó phải dùng long syntax với `condition: service_healthy`.

**4. Compose tự tạo network như thế nào nếu không khai báo `networks:`?**
Tự tạo một network kiểu `bridge` tên `<project_name>_default`, gắn mọi service trong file vào đó, tự cấu hình DNS nội bộ để service gọi nhau bằng tên.

**5. `docker compose down` và `docker compose down -v` khác nhau thế nào?**
`down` xóa container và network nhưng giữ nguyên named volume. `down -v` xóa cả named volume — mất dữ liệu nếu không có backup, cần cẩn trọng tuyệt đối khi dùng trên production.

## Mid-level

**6. Giải thích cơ chế project name và vì sao hai project Compose khác nhau không đụng nhau dù dùng cùng tên service.**
Project name (mặc định suy ra từ tên thư mục chứa compose.yaml, hoặc chỉ định qua `-p`/`COMPOSE_PROJECT_NAME`/khóa `name:`) được dùng làm tiền tố cho mọi resource thật: container (`<project>-<service>-<n>`), network mặc định (`<project>_default`), volume (`<project>_<tên>`). Hai project khác thư mục có project name khác nhau, nên dù cả hai đều khai báo service tên `db`, container/network/volume thật của chúng vẫn hoàn toàn tách biệt.

**7. Giải thích 3 giá trị của `depends_on.condition` và khi nào dùng mỗi loại.**
`service_started` (mặc định) — chỉ chờ container được start, phù hợp khi dependency không cần thời gian khởi tạo. `service_healthy` — chờ container đạt trạng thái `healthy` theo healthcheck, dùng cho database/service cần thời gian sẵn sàng thực sự. `service_completed_successfully` — chờ container (thường job một lần như migration) exit với code 0, dùng khi cần một bước chạy xong hoàn toàn trước khi bước tiếp theo bắt đầu (ví dụ chạy migration xong mới start backend).

**8. Vì sao dùng file-based secrets tốt hơn đặt password trực tiếp trong `environment:`?**
`environment:` với giá trị hardcode dễ bị commit vào Git, và có thể đọc ngược lại qua `docker inspect` (biến môi trường container hiển thị trong metadata). File-based secrets mount nội dung file vào `/run/secrets/<name>` với quyền đọc giới hạn, không xuất hiện trong biến môi trường/metadata container, và file thật (`./secrets/*.txt`) không commit vào Git.

**9. Compose quyết định có cần recreate container khi chạy lại `up` hay không dựa vào đâu?**
Dựa vào so sánh hash cấu hình (lưu ở label `com.docker.compose.config-hash` gắn vào container lúc tạo) giữa cấu hình hiện tại trong compose.yaml và cấu hình lúc container được tạo lần trước. Nếu hash không đổi, Compose không recreate — đây là lý do một số thay đổi (như sửa nội dung file bind mount mà không đổi đường dẫn) không kích hoạt recreate.

**10. `profiles` giải quyết vấn đề gì trong một compose.yaml thực tế của doanh nghiệp?**
Cho phép tách các service phụ trợ (admin UI, debug tool, seed data job) ra khỏi luồng chạy mặc định — chúng không tự động chạy khi `docker compose up -d` thông thường, chỉ chạy khi chỉ định tường minh `--profile <tên>` hoặc biến `COMPOSE_PROFILES`. Giúp một file compose.yaml duy nhất phục vụ được nhiều mục đích (dev đầy đủ công cụ debug, production tối giản) mà không cần duy trì nhiều file trùng lặp.

## Thực chiến

**11. Sau khi deploy một compose.yaml mới lên staging, `backend` báo lỗi kết nối `db` liên tục dù `healthcheck` của `db` đã báo `healthy` từ lâu trước khi `backend` start. Hướng điều tra tiếp theo là gì, ngoài vấn đề `depends_on`?**
Vì `depends_on.condition: service_healthy` đã đúng và `db` đã healthy trước, vấn đề không còn nằm ở thứ tự khởi động mà ở tầng network hoặc cấu hình kết nối: kiểm tra `backend` và `db` có cùng nằm trong network đã khai báo hay không (dễ sai khi có nhiều network tách biệt frontend/backend), kiểm tra biến môi trường `DB_HOST` có đúng tên service `db` hay bị hardcode nhầm IP cũ từ lần chạy trước, và kiểm tra healthcheck của `db` có thực sự kiểm tra đúng điều kiện backend cần (ví dụ `pg_isready` chỉ xác nhận Postgres accept connection TCP, không đảm bảo database/schema cụ thể mà backend cần đã tồn tại — nếu backend cần một database cụ thể chưa được tạo, kết nối TCP vẫn thành công nhưng query thất bại, dễ nhầm là lỗi "connection" trong khi thực ra là lỗi "database chưa sẵn sàng ở mức logic").

**12. Team yêu cầu compose.yaml production không được chứa bất kỳ `build:` nào, chỉ dùng `image:` trỏ registry nội bộ. Giải thích lý do kiến trúc đằng sau yêu cầu này, và cách bạn vẫn giữ được một compose.yaml dùng chung cho cả dev và production.**
`build:` trong compose.yaml production nghĩa là mỗi lần `docker compose up` trên server production có thể build lại image tại chỗ — không đảm bảo tính nhất quán (image build trên server production có thể khác image đã qua CI/CD test do thời điểm build khác, dependency version có thể trôi), và tốn tài nguyên/thời gian không cần thiết trên node production. Cách giữ một file dùng chung: dùng biến môi trường/`.env` cho `image:` (ví dụ `image: myapp-backend:${IMAGE_TAG:-latest}`), dev override bằng file `compose.override.yaml` (Compose tự động merge file này nếu tồn tại) thêm `build:` chỉ cho môi trường dev, production không có file override đó nên chỉ dùng đúng `image:` đã build sẵn từ registry.

**13. Một service trong compose.yaml cần đọc secret nhưng bản thân ứng dụng (tự viết, không phải image chính thức như Postgres) không hỗ trợ quy ước `_FILE` suffix. Bạn xử lý thế nào mà vẫn tuân thủ nguyên tắc không hardcode secret vào `environment:`?**
Có hai hướng: (1) sửa code ứng dụng để tự đọc trực tiếp từ đường dẫn cố định `/run/secrets/<tên_secret>` lúc khởi động (cách bền vững nhất, phù hợp nếu bạn kiểm soát được source code); (2) nếu không thể sửa code ngay, dùng một entrypoint script wrapper trong image: script đọc nội dung file secret tại `/run/secrets/<tên>`, gán vào biến môi trường thực sự ngay trước khi `exec` chạy ứng dụng chính (biến môi trường lúc này chỉ tồn tại trong chính tiến trình runtime, không nằm trong `environment:` của compose.yaml nên không bị commit vào Git hay lộ qua cấu hình tĩnh, dù vẫn xuất hiện trong `docker inspect` nếu ai đó soi kỹ tại runtime — chấp nhận được như giải pháp tạm trong lúc chờ sửa ứng dụng đúng chuẩn).

**14. Một stack Compose có 6 service, mỗi lần `docker compose up -d` mất hơn 2 phút vì service cuối cùng phải chờ tuần tự qua toàn bộ dependency graph dù nhiều service không thực sự phụ thuộc nhau. Bạn tối ưu thứ tự khởi động thế nào mà vẫn đảm bảo đúng logic phụ thuộc?**
Rà soát lại từng `depends_on` — nếu một số service được khai báo phụ thuộc "cho chắc" nhưng thực chất không có quan hệ dữ liệu/kết nối thật (ví dụ `nginx` bị khai `depends_on: [db]` dù nó chỉ gọi `backend`, không bao giờ gọi thẳng `db`), gỡ bỏ các phụ thuộc thừa đó để dependency graph thực sự phản ánh đúng quan hệ cần thiết — Compose tự động chạy song song mọi service không có quan hệ phụ thuộc trực tiếp/gián tiếp với nhau, chỉ những service thực sự có `depends_on` mới bị tuần tự hóa. Đồng thời rà soát `start_period`/`interval` của healthcheck có đang đặt quá dài một cách không cần thiết (ví dụ `interval: 10s` cho một service khởi động chỉ mất 1-2 giây thực tế) — giảm xuống mức hợp lý để Compose phát hiện `healthy` sớm hơn mà không cần tăng rủi ro false positive.
