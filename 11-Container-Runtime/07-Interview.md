---
title: "07 - Interview"
module: 11
tags: [docker, sysops-infra, module-11, interview]
---

# 07 - Interview: Container Runtime

## Junior

**1. `docker stop` và `docker kill` khác nhau thế nào?**
`stop` gửi SIGTERM, chờ grace period (mặc định 10 giây) cho tiến trình tự thoát, hết giờ mới SIGKILL. `kill` gửi SIGKILL ngay lập tức, không grace period, không cơ hội dọn dẹp.

**2. Exit code 137 nghĩa là gì?**
137 = 128 + 9, trong đó 9 là số hiệu SIGKILL — container bị buộc dừng cứng, không graceful. Có thể do OOM kill hoặc do `docker kill`/timeout sau `docker stop`. Xác nhận nguyên nhân chính xác bằng `docker inspect --format '{{.State.OOMKilled}}'`.

**3. `docker exec` và `docker attach` khác nhau thế nào, cái nào an toàn hơn để debug?**
`exec` tạo tiến trình mới độc lập bên trong namespace container, không đụng chạm tiến trình chính — an toàn. `attach` kết nối trực tiếp vào stdin/stdout của PID 1, rủi ro vô tình gửi signal (như Ctrl+C) làm dừng container. Luôn ưu tiên `exec` để debug.

**4. Bốn restart policy của Docker là gì?**
`no` (mặc định, không tự khởi động lại), `on-failure[:N]` (chỉ khi exit code khác 0, giới hạn N lần thử), `always` (luôn tự khởi động lại kể cả sau khi bạn `docker stop`, trừ khi daemon nhớ trạng thái dừng thủ công), `unless-stopped` (giống always nhưng không tự bật lại nếu bạn chủ động dừng).

**5. `docker system prune` làm gì?**
Dọn dẹp tài nguyên không dùng: container đã exited, network không container nào dùng, dangling image, build cache. Cần cẩn trọng với `-a` (mở rộng xóa cả image không có container dùng) và `--volumes` (xóa cả volume không tham chiếu) trong production.

## Mid-level

**6. Giải thích vì sao PID 1 trong container "đặc biệt" về xử lý signal, và hệ quả thực tế.**
Kernel Linux không áp dụng hành vi mặc định của một số signal (như SIGTERM) cho PID 1 trừ khi tiến trình tự đăng ký handler xử lý signal đó — đây là quy tắc riêng của kernel dành cho PID 1. Nếu ứng dụng làm PID 1 trong container mà không tự bắt SIGTERM, `docker stop` sẽ không có tác dụng dừng nhẹ nhàng, phải đợi hết grace period rồi mới SIGKILL — gây chậm deploy và mất khả năng graceful shutdown.

**7. `--memory` và `--memory-reservation` khác nhau thế nào?**
`--memory` là giới hạn cứng (map tới `memory.max` trong cgroups v2) — vượt là bị OOM kill ngay. `--memory-reservation` là ngưỡng mềm (map tới `memory.low`) — chỉ có tác dụng khi toàn host chịu áp lực bộ nhớ, kernel cố gắng ép container về dưới ngưỡng này trước khi phải OOM kill container khác.

**8. Vì sao cần `--init` cho container chạy shell script hoặc có khả năng fork tiến trình con?**
Nếu không có `--init`, ứng dụng/script làm PID 1 phải tự chịu trách nhiệm reap zombie process của các tiến trình con đã kết thúc — hầu hết ứng dụng không tự làm việc này, dẫn đến zombie tích lũy theo thời gian, chiếm slot process table. `--init` chèn `tini` làm PID 1 thật, tự động reap zombie và forward signal đúng chuẩn xuống tiến trình ứng dụng con.

**9. `docker events` khác `docker logs` ở điểm nào, dùng khi nào?**
`docker logs` xem output (stdout/stderr) của một container cụ thể. `docker events` stream các sự kiện ở cấp daemon (start/stop/die/oom, pull image, network connect...) trên toàn hệ thống, không giới hạn một container — dùng khi cần quan sát bức tranh tổng thể "chuyện gì đang xảy ra" khi debug sự cố liên quan nhiều container, hoặc tích hợp vào hệ thống alerting riêng.

**10. Giải thích cơ chế quota/period của `--cpus` trong cgroups v2.**
Kernel chia thời gian thành các chu kỳ nhỏ (period, mặc định 100ms) và giới hạn tổng thời gian CPU container được dùng trong mỗi chu kỳ (quota) tương ứng giá trị `--cpus`. Ví dụ `--cpus=1.5` nghĩa là container được dùng tối đa 150ms CPU-time trong mỗi 100ms period. Đây không phải giới hạn cứng theo phần cứng mà là giới hạn theo tỷ lệ thời gian được lập lịch.

## Thực chiến

**11. Một service production dùng `--restart=always`, sau khi bạn `docker stop` để bảo trì, 30 giây sau nó tự chạy lại. Giải thích và đề xuất sửa quy trình bảo trì.**
Đây đúng hành vi của `always` — Docker chỉ "nhớ" trạng thái dừng thủ công tạm thời trong phiên hoạt động hiện tại của daemon nhưng nhiều trường hợp/phiên bản có thể tự khởi động lại sớm hơn mong đợi nếu có sự kiện kích hoạt (ví dụ healthcheck orchestrator bên ngoài, hoặc daemon restart xen giữa). Khắc phục quy trình: với service cần bảo trì có kiểm soát, nên đổi sang `unless-stopped` (đảm bảo dừng thủ công thực sự giữ nguyên trạng thái dừng), hoặc dùng quy trình bảo trì chuẩn hơn (đưa ra khỏi load balancer trước, cập nhật `--restart=no` tạm thời qua `docker update` trong lúc bảo trì, khôi phục sau).

**12. Container database bị OOM kill mỗi ngày vào đúng khung giờ cao điểm, dù `--memory` đã set gấp đôi RSS trung bình đo được lúc bình thường. Hướng điều tra tiếp theo là gì?**
RSS trung bình không phản ánh đỉnh sử dụng bộ nhớ lúc cao điểm (query nặng, cache buffer pool tăng đột biến, connection pool phình to). Cần theo dõi `docker stats` liên tục (không chỉ snapshot) hoặc export dữ liệu memory.current theo thời gian qua cgroup để thấy đúng đỉnh, đối chiếu với thời điểm OOM. Với database cụ thể, kiểm tra cấu hình nội bộ (ví dụ buffer pool size của MySQL/Postgres) có đang set vượt quá giới hạn cgroup container hay không — đây là lỗi cấu hình rất phổ biến: ứng dụng tự tính toán bộ nhớ dựa trên RAM tổng của host (`/proc/meminfo`) thay vì giới hạn cgroup thực tế của chính nó.

**13. Sau khi thêm `HEALTHCHECK` và orchestrator dựa vào đó để route traffic, một container "treo" (deadlock, không phản hồi) nhưng vẫn hiển thị `Up` trong `docker ps` suốt nhiều giờ trước khi ai đó phát hiện. Vì sao restart policy không tự xử lý được tình huống này, và giải pháp đúng là gì?**
Restart policy chỉ kích hoạt khi tiến trình chính **thoát hẳn** (process exit) — một tiến trình bị deadlock vẫn "sống" theo nghĩa kernel (không thoát), nên `docker ps` vẫn báo `Up`, restart policy không có gì để phản ứng. Giải pháp đúng là `HEALTHCHECK` phải thực sự kiểm tra được tình trạng "sẵn sàng phục vụ" (không chỉ "process còn sống"), và cần một cơ chế bên ngoài (script giám sát, hoặc ở tầng orchestrator cao hơn như Kubernetes liveness probe sẽ học sau) chủ động `docker restart` khi phát hiện trạng thái `unhealthy` kéo dài — bản thân Docker Engine đơn lẻ không tự động restart container chỉ vì healthcheck báo unhealthy, đây là điểm dễ hiểu lầm.

**14. Bạn được giao dọn dẹp một build server có `/var/lib/docker` chiếm 180GB nhưng chỉ có 5 image đang thực sự dùng bởi container chạy. Quy trình dọn dẹp an toàn dùng những lệnh nào, theo thứ tự nào?**
Đầu tiên `docker system df -v` để biết phân bổ dung lượng theo từng loại (images, containers, volumes, build cache) — thường build cache chiếm phần lớn trên build server. Dọn theo thứ tự an toàn nhất trước: `docker container prune` (chỉ xóa container đã exited, an toàn tuyệt đối), `docker builder prune --all --force` (build cache, an toàn vì có thể build lại), `docker image prune -a` (xóa image không container nào tham chiếu — cần xác nhận trước danh sách vì có thể có image được pull sẵn để dùng sau). Cuối cùng mới cân nhắc `docker volume prune` sau khi đã kiểm tra kỹ danh sách volume — đây là bước rủi ro nhất vì volume chứa dữ liệu, không tái tạo lại được như image/cache.
