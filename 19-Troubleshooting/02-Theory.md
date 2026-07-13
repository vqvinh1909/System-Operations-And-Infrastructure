---
title: "02 - Theory"
module: 19
tags: [docker, sysops-infra, module-19, troubleshooting, debugging]
---

# 02 — Lý thuyết: Mô hình Debug Phân Tầng

## 1. Vì sao cần một mô hình phân tầng thay vì "thử hết mọi thứ"

Một sự cố Docker luôn biểu hiện ra ngoài ở **một tầng**, nhưng nguyên nhân gốc rễ có thể nằm ở **một tầng hoàn toàn khác**. Ví dụ kinh điển: ứng dụng trả lỗi 502 (biểu hiện ở tầng Application), nhưng nguyên nhân thật là container database bị OOM-killed (nguyên nhân ở tầng Performance/Host) vài giây trước đó. Nếu bạn chỉ nhìn vào tầng biểu hiện (sửa code ứng dụng, restart container web), sự cố sẽ tái diễn vì gốc rễ chưa được chạm tới.

Mô hình debug phân tầng chia hệ thống thành 5 lớp rõ ràng, mỗi lớp có bộ câu hỏi và bộ lệnh chẩn đoán riêng:

```
Application  ->  Container  ->  Network  ->  Storage  ->  Host/Kernel
```

Nguyên tắc áp dụng: **đi từ ngoài vào trong, loại trừ từng tầng một** bằng bằng chứng cụ thể (log, số liệu, kết quả lệnh) chứ không phải phỏng đoán — chỉ chuyển sang tầng tiếp theo khi đã có bằng chứng rõ ràng tầng hiện tại không phải nguyên nhân.

## 2. Tầng Application — luôn kiểm tra trước tiên

Trước khi nghi ngờ Docker, xác nhận vấn đề không nằm ở chính logic ứng dụng: lỗi trả về là lỗi nghiệp vụ (400, 422 — do dữ liệu đầu vào) hay lỗi hạ tầng (502, 503, timeout — do tầng bên dưới không phản hồi được)? Đây là bước rẻ nhất nhưng thường bị bỏ qua khi vội — nhiều "sự cố Docker" hóa ra chỉ là lỗi logic ứng dụng bình thường bị nhầm lẫn vì xảy ra đúng lúc có thay đổi hạ tầng gần đây.

## 3. Tầng Container — vòng đời và trạng thái

Câu hỏi cốt lõi: **container có đang chạy đúng như kỳ vọng không?** Ba trạng thái cần phân biệt rõ:

- **Không khởi động được** (`Exited` ngay lập tức, exit code khác 0) — thường là lỗi cấu hình, entrypoint sai, thiếu biến môi trường bắt buộc.
- **Restart loop** (`Restarting` liên tục) — container khởi động được nhưng crash sau vài giây, lặp lại vô hạn theo `restart policy`. Cực kỳ dễ nhầm với "không khởi động được" nếu không nhìn kỹ `docker ps` (trạng thái `Restarting` khác `Exited`).
- **Đang chạy nhưng không phản hồi đúng** — container ở trạng thái `Up`, health check (nếu có cấu hình) báo `unhealthy`, hoặc healthy nhưng ứng dụng bên trong vẫn treo (trường hợp khó nhất, cần vào tận bên trong container để kiểm tra).

**Exit code** là manh mối quan trọng bị bỏ qua nhiều nhất — mỗi mã lỗi kể một câu chuyện khác nhau:

| Exit code | Ý nghĩa thường gặp |
|---|---|
| `0` | Tiến trình chính kết thúc bình thường (nếu không mong muốn container tự dừng, kiểm tra lại logic entrypoint) |
| `1` | Lỗi ứng dụng chung chung — cần xem log để biết chi tiết |
| `137` | Bị kill bằng tín hiệu `SIGKILL` (128 + 9) — rất thường gặp do **OOM Killer** của kernel, xem tầng Performance |
| `139` | Segmentation fault (128 + 11) — lỗi nghiêm trọng ở tầng runtime/thư viện native |
| `143` | Bị dừng bằng `SIGTERM` (128 + 15) — thường là dừng có chủ đích (`docker stop`), không phải lỗi |

## 4. Tầng Network — mô hình "từ trong ra ngoài"

Sự cố network trong Docker luôn nên chẩn đoán theo thứ tự từ gần tới xa:

1. **Bên trong container có tự hoạt động không** — tiến trình có đang lắng nghe đúng port không (`netstat`/`ss` bên trong container).
2. **Container này có gọi được container khác trong cùng network không** — kiểm tra DNS nội bộ của Docker (`127.0.0.11`), kiểm tra hai container có chung một network không.
3. **Từ host có gọi được container qua port mapping không** — kiểm tra port publish đúng chưa, có xung đột port với tiến trình khác trên host không.
4. **Từ bên ngoài host có gọi được không** — kiểm tra firewall host, security group (nếu chạy cloud), routing.

Đi ngược thứ tự này (bắt đầu kiểm tra từ bên ngoài) thường lãng phí thời gian vì bạn không biết chắc vấn đề có thật sự ở tầng ngoài hay chỉ là hệ quả của lỗi bên trong.

## 5. Tầng Storage — ba loại vấn đề khác nhau, dễ nhầm lẫn với nhau

- **Permission**: container ghi/đọc bị từ chối dù bind mount đúng đường dẫn — nguyên nhân thường là UID bên trong container khác UID sở hữu thư mục trên host (đặc biệt phổ biến khi ứng dụng chạy bằng non-root user theo đúng khuyến nghị bảo mật ở Module 18).
- **Dung lượng**: đĩa host đầy (không chỉ do log như đã học ở Module 17 — còn do image layer tích lũy, volume không dọn, container đã dừng nhưng chưa xóa).
- **Driver/Integrity**: lỗi ở chính overlay2 storage driver (hiếm hơn nhưng nghiêm trọng hơn) — thường biểu hiện qua lỗi lạ khi container cố ghi file, không liên quan trực tiếp tới quyền hay dung lượng.

## 6. Tầng Performance — phân biệt "giới hạn cấu hình sai" và "host thật sự quá tải"

Đây là tầng dễ chẩn đoán sai nhất, vì hai nguyên nhân sau tạo ra triệu chứng bề ngoài giống hệt nhau (ứng dụng chậm/bị kill):

- **Giới hạn cấu hình sai**: bạn đặt `--memory=256m` cho một ứng dụng cần 512MB lúc tải cao — container sẽ bị OOM-killed dù host còn rất nhiều RAM trống. Đây là lỗi cấu hình, sửa bằng cách tăng limit hoặc tối ưu ứng dụng, không phải vấn đề của host.
- **Host thật sự quá tải**: tổng nhu cầu tài nguyên của tất cả container trên host vượt quá tài nguyên vật lý thật sự có — dù từng container cấu hình limit hợp lý, tranh chấp tài nguyên (resource contention) vẫn xảy ra ở tầng kernel/scheduler.

Cách phân biệt: đọc **cgroup của chính container** (biết được nó có đang chạm limit riêng của nó không) **và** đọc tài nguyên tổng của host (`vmstat`, `top`) cùng lúc — chỉ dựa vào một trong hai nguồn dữ liệu dễ dẫn tới kết luận sai. Xem chi tiết lệnh cụ thể ở [[04-Commands]] và các kịch bản đầy đủ ở [[06-Troubleshooting]].

## 7. Nguyên tắc chung xuyên suốt cả 5 tầng

1. **Luôn xác nhận bằng bằng chứng, không phỏng đoán** — mỗi bước chuyển tầng phải dựa trên kết quả một lệnh cụ thể, không phải cảm giác "chắc là do...".
2. **Ghi lại timeline** — đối chiếu thời điểm sự cố bắt đầu với lịch sử deploy/thay đổi cấu hình gần nhất (`docker events`, lịch sử registry ở Module 16) thường rút ngắn đáng kể thời gian điều tra.
3. **Tái tạo được sự cố là một nửa của việc sửa được nó** — nếu có thể tái tạo sự cố trong môi trường lab an toàn (như các Lab ở [[05-Labs]]), việc chẩn đoán trở nên nhanh và chắc chắn hơn nhiều so với chỉ suy luận trên production đang cháy.
