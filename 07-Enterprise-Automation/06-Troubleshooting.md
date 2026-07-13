---
title: "06 - Troubleshooting"
module: 7
tags: [ansible, sysops-infra, module-07, troubleshooting, rolling-update]
---

# 06 — Troubleshooting

## Lỗi 1: Rolling Update dừng giữa chừng, một số host đã cập nhật, số khác chưa

**Triệu chứng:** Playbook báo dừng do vượt `max_fail_percentage`, hạ tầng ở trạng thái "nửa cũ nửa mới".

**Đây là hành vi có chủ đích, không phải lỗi** — `max_fail_percentage` được thiết kế để dừng sớm, giới hạn phạm vi ảnh hưởng. Tuy nhiên, trạng thái "nửa cũ nửa mới" có thể tự nó là rủi ro nếu 2 phiên bản không tương thích (ví dụ thay đổi schema database mà phiên bản cũ của ứng dụng không đọc được).

**Cách xử lý:**
1. Điều tra ngay nguyên nhân host thất bại (log `-vvv` của các host lỗi) trước khi quyết định bước tiếp theo.
2. Nếu 2 phiên bản không tương thích, ưu tiên rollback ngay các host đã cập nhật về phiên bản cũ (chạy lại Playbook với biến `app_version` trỏ về bản trước) thay vì cố chạy tiếp.
3. Thiết kế phòng ngừa: đảm bảo mọi thay đổi ứng dụng/database tuân thủ nguyên tắc **tương thích ngược tạm thời** (backward compatible) trong suốt quá trình rolling update — kỹ thuật tiêu chuẩn trong triển khai doanh nghiệp thực tế, không riêng gì Ansible.

## Lỗi 2: Health check luôn báo thành công dù ứng dụng thực sự chưa sẵn sàng

**Triệu chứng:** node được đưa lại vào pool Load Balancer nhưng người dùng vẫn gặp lỗi 502/503 ngay sau đó.

**Nguyên nhân thường gặp:** endpoint `/healthz` chỉ kiểm tra "web server đang chạy" (ví dụ chỉ trả về HTTP 200 tĩnh) mà không thực sự kiểm tra các phụ thuộc quan trọng (kết nối database, cache, service phụ thuộc khác) — health check "giả" (shallow health check).

**Cách xử lý:** thiết kế `/healthz` kiểm tra thật các phụ thuộc quan trọng nhất (ví dụ thử kết nối database timeout ngắn), chấp nhận đánh đổi health check chậm hơn một chút để đảm bảo độ tin cậy thật.

## Lỗi 3: `delegate_to` chạy nhầm trên host đang được vòng lặp, không phải host đích

**Triệu chứng:** Task điều khiển HAProxy (đáng lẽ chạy trên Load Balancer) lại chạy trên chính web server đang được cập nhật.

**Nguyên nhân:** cú pháp `delegate_to` sai, hoặc dùng biến `{{ inventory_hostname }}` thay vì đúng biến trỏ tới Load Balancer (`groups['loadbalancers'][0]`).

**Cách xử lý:**
```bash
# Chay voi -vvv, kiem tra dong "delegated to" trong log co dung host mong doi khong
ansible-playbook zero-downtime.yml -vvv | grep -i delegat
```

## Lỗi 4: Backup báo thành công nhưng file backup rỗng hoặc hỏng

**Triệu chứng:** Task `stat` xác nhận file tồn tại, nhưng khi thử restore, file backup không dùng được.

**Nguyên nhân:** kiểm tra `size == 0` chỉ phát hiện file **hoàn toàn rỗng**, không phát hiện file bị cắt ngắn (truncated) do lệnh backup bị ngắt giữa chừng (hết dung lượng đĩa, timeout) nhưng vẫn tạo ra một file có kích thước khác 0.

**Cách xử lý:** với backup quan trọng, thêm bước kiểm tra sâu hơn — ví dụ với `pg_dump`, chạy thử `pg_restore --list` trên file backup để xác nhận cấu trúc hợp lệ, không chỉ dựa vào kích thước file. Với chiến lược trưởng thành hơn, định kỳ test khôi phục thật (restore drill) trên môi trường riêng, không chỉ tin vào "backup chạy không báo lỗi".

## Lỗi 5: `run_once` gửi thông báo monitoring nhưng chạy trên sai host

**Triệu chứng:** Task `run_once: true` chạy nhưng dùng biến/facts của một host ngẫu nhiên trong nhóm, không phải host mong muốn.

**Nguyên nhân:** `run_once` chỉ đảm bảo Task chạy **một lần**, nhưng theo mặc định Task đó vẫn thực thi trên **host đầu tiên** của danh sách host trong Play (không nhất thiết là host bạn kỳ vọng), trừ khi kết hợp `delegate_to`.

**Cách xử lý:** luôn kết hợp `run_once: true` với `delegate_to: localhost` (hoặc một host cụ thể) khi Task cần chạy trên đúng một máy xác định (ví dụ control node để gọi API monitoring) — không dựa vào hành vi mặc định "host đầu tiên trong danh sách", vì thứ tự này có thể thay đổi tùy thuộc cách inventory được resolve.
