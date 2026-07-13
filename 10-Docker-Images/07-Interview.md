---
title: "07 - Interview"
module: 10
tags: [docker, sysops-infra, module-10, interview]
---

# 07 - Interview: Docker Images

## Junior

**1. Sự khác biệt giữa `COPY` và `ADD`?**
Cả hai copy file/thư mục từ build context vào image. `ADD` có thêm khả năng tự giải nén archive local và tải file từ URL — hành vi "ẩn", ít minh bạch. Khuyến nghị luôn ưu tiên `COPY`, chỉ dùng `ADD` khi thực sự cần auto-extract archive local.

**2. `ENTRYPOINT` và `CMD` khác nhau thế nào?**
`ENTRYPOINT` là lệnh cố định luôn chạy. `CMD` là tham số mặc định, có thể bị ghi đè bởi argument truyền vào lúc `docker run image <args>`. Kết hợp phổ biến: `ENTRYPOINT ["nginx"]` + `CMD ["-g", "daemon off;"]`.

**3. `ARG` và `ENV` khác nhau thế nào?**
`ARG` chỉ tồn tại lúc build, set qua `--build-arg`, không truy cập được từ container đang chạy. `ENV` tồn tại cả lúc build và lúc container chạy, truy cập được qua `docker exec ... env`.

**4. Vì sao không nên dùng tag `latest` trong production?**
`latest` không đảm bảo tính bất biến — ai đó push đè `latest` là nội dung image thay đổi hoàn toàn dù tên tag không đổi, gây rủi ro deploy nhầm phiên bản. Production nên pin version cụ thể (semver hoặc Git SHA).

**5. `EXPOSE` trong Dockerfile có tự động mở port ra ngoài không?**
Không. `EXPOSE` chỉ là metadata/tài liệu khai báo ứng dụng lắng nghe port nào. Port chỉ thực sự publish ra host khi chạy `docker run -p <host-port>:<container-port>`.

## Mid-level

**6. Giải thích cơ chế cache của Docker layer, và nguyên tắc sắp xếp Dockerfile để tối ưu cache.**
Mỗi instruction tạo một layer, cache key tính từ (instruction + tham số) và cache key của layer cha; với `COPY`/`ADD` còn tính thêm checksum nội dung file. Nếu layer cha miss, mọi layer con đều miss theo (hiệu ứng domino). Nguyên tắc: đặt các bước ít thay đổi (FROM, cài dependency theo manifest) trước, đặt `COPY . .` (source code, đổi thường xuyên) sau cùng.

**7. Vì sao cần multi-stage build? Cho ví dụ cụ thể.**
Tách phần chỉ cần lúc build (compiler, dev dependency, source code) khỏi phần cần lúc chạy (binary/artifact cuối). Ví dụ: stage `builder` dùng `golang:1.23` compile ra binary tĩnh, stage `final` dùng `alpine:3.20` chỉ `COPY --from=builder` đúng 1 file binary — giảm image từ ~900MB xuống ~15MB, đồng thời loại bỏ source code và compiler khỏi image production.

**8. `RUN` với shell form và exec form khác nhau thế nào, ảnh hưởng gì tới graceful shutdown?**
Shell form (`RUN lệnh`) chạy qua `/bin/sh -c`, tiến trình ứng dụng trở thành con của shell, không phải PID 1. Exec form (`RUN ["lệnh"]`, tương tự với CMD/ENTRYPOINT) chạy trực tiếp, ứng dụng là PID 1. Với CMD/ENTRYPOINT dùng shell form, khi Docker gửi SIGTERM tới PID 1 (là `/bin/sh`), tín hiệu không tự động forward tới tiến trình ứng dụng con, khiến container mất thời gian (grace period) rồi mới bị SIGKILL — nên ưu tiên exec form.

**9. BuildKit cache mount khác gì so với layer cache thông thường?**
Layer cache chỉ HIT khi input (instruction + file liên quan) không đổi. Cache mount (`--mount=type=cache,target=...`) tạo một vùng lưu trữ riêng biệt do BuildKit quản lý, tồn tại xuyên suốt nhiều lần build **độc lập với layer cache** — luôn khả dụng kể cả khi source code đổi liên tục, không làm tăng kích thước image vì không nằm trong layer nào. Dùng cho cache package manager (apt, pip, npm, Go module, Maven).

**10. Vì sao dùng `ARG`/`ENV` để truyền secret là nguy hiểm? Cách làm đúng?**
Giá trị `ARG`/`ENV` vẫn lưu lại trong build history/metadata của image (xem được bằng `docker history`), dù `ARG` không tồn tại lúc runtime — coi như secret đã bị lộ công khai cho bất kỳ ai có quyền pull image. Cách đúng: dùng BuildKit secret mount (`--mount=type=secret`), secret chỉ tồn tại trong bộ nhớ tạm đúng lúc `RUN` cần, không bao giờ ghi vào layer.

## Thực chiến

**11. Team dev báo image build ra 1.2GB cho một service Python chỉ có vài trăm dòng code. Bạn kiểm tra và xử lý theo quy trình nào?**
Chạy `docker history --no-trunc` để xác định layer nào chiếm dung lượng lớn nhất (thường là layer cài dependency hoặc base image không phù hợp). Kiểm tra base image có đang dùng `python:3.12` đầy đủ (Debian full, ~900MB) thay vì `python:3.12-slim` (~150MB) hay không. Kiểm tra RUN có gộp cài + dọn cache trong cùng layer không (`pip install` để lại `__pycache__`, cache pip nếu không dùng `--no-cache-dir` hoặc dọn sau). Nếu có bước build native extension (ví dụ compile C extension), cân nhắc multi-stage tách builder ra khỏi final.

**12. Sau khi thêm `HEALTHCHECK` vào Dockerfile, container luôn báo `unhealthy` ngay cả khi ứng dụng chạy bình thường, dù test thủ công endpoint `/health` trả 200 OK. Chẩn đoán thế nào?**
Kiểm tra `--start-period` có đủ dài cho thời gian khởi động thật của ứng dụng không (ví dụ JVM warm-up lâu hơn `start-period` mặc định, mỗi lần check trong giai đoạn khởi động đều fail và bị tính vào `--retries`). Kiểm tra lệnh healthcheck có chạy được **bên trong** container hay không — nhiều image tối giản (Alpine, distroless) không có `curl`, chỉ có `wget` hoặc không có gì cả, khiến chính câu lệnh healthcheck bị lỗi "command not found" chứ không phải ứng dụng thật sự unhealthy. Test bằng `docker exec <container> <đúng lệnh trong HEALTHCHECK>` để xác nhận.

**13. Một pipeline CI build image xong, push lên registry, nhưng lần deploy sau đó chạy image cũ dù build log báo thành công. Nguyên nhân kiến trúc nào liên quan tới image tagging cần kiểm tra?**
Khả năng cao pipeline dùng tag cố định (`latest` hoặc tag không đổi giữa các lần build) mà bước deploy pull image dựa vào tag đó — nếu node đích đã có sẵn image local cùng tag từ lần trước, và cấu hình pull policy không ép `always pull`/không kiểm tra digest, node có thể dùng lại image cache cũ thay vì pull bản mới. Khắc phục: tag theo Git SHA hoặc build number duy nhất cho mỗi lần build, đảm bảo pull policy luôn kiểm tra digest mới nhất, hoặc pin theo digest cụ thể (`image@sha256:...`) trong bước deploy.

**14. Security team yêu cầu chứng minh mọi image production đều build từ Dockerfile đã review, không có ai build thủ công từ máy cá nhân rồi push thẳng. Bạn thiết kế kiểm soát nào dựa trên kiến thức Image Layer/LABEL đã học?**
Bắt buộc mọi image production build qua CI/CD pipeline (không cho phép credential push trực tiếp từ máy cá nhân vào registry production), gắn `LABEL` ghi rõ Git commit SHA và CI build ID vào mỗi image lúc build, và có thể ký số image (image signing, ví dụ Docker Content Trust hoặc cosign — nội dung nâng cao hơn ở Module Registry) để registry/orchestrator từ chối chạy image không có chữ ký hợp lệ. Đối chiếu `LABEL` bằng `docker inspect`/registry API để audit nguồn gốc bất kỳ image nào đang chạy trên production, truy ngược đúng commit và pipeline đã tạo ra nó.
