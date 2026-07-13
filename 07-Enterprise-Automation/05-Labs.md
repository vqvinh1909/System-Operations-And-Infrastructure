---
title: "05 - Labs"
module: 7
tags: [ansible, sysops-infra, module-07, labs]
---

# 05 — Labs thực hành

> Yêu cầu môi trường: mở rộng inventory hiện có thành 3 tầng — 1 Load Balancer (cài HAProxy), 3-4 Web server (Role `nginx_hardened`/ứng dụng giả lập từ Module 06), 1 Database server. Có thể dùng VM/container cho tất cả nếu tài nguyên hạn chế.

## Lab 1 — Dựng kiến trúc Multi-tier cơ bản

**Mục tiêu:** có đủ 3 tầng chạy được, Load Balancer phân phối traffic tới Web tier.

**Các bước:**

1. Cập nhật inventory: thêm nhóm `loadbalancers` (1 host), `webservers` (3-4 host), `dbservers` (1 host).
2. Viết Role `haproxy_lb` tối giản: cài HAProxy, render `haproxy.cfg.j2` liệt kê toàn bộ host trong `groups['webservers']` (dùng kỹ thuật `{% for %}` + `hostvars` ở [[04-Commands|04 - Commands]] mục 5).
3. Trên mỗi web server, đảm bảo có endpoint `/healthz` trả về HTTP 200 (có thể dùng file tĩnh đơn giản qua nginx nếu chưa có ứng dụng thật).
4. Chạy Playbook triển khai cả 3 tầng, xác nhận truy cập qua IP Load Balancer phân phối luân phiên tới các web server (`curl` nhiều lần, quan sát response khác nhau nếu mỗi web server trả về hostname riêng).

**Kết quả cần có:** cấu trúc inventory 3 nhóm, Role `haproxy_lb`, log truy cập luân phiên qua Load Balancer.

## Lab 2 — Rolling Update với `serial`

**Mục tiêu:** cập nhật Web tier theo từng đợt, không phải toàn bộ cùng lúc.

**Các bước:**

1. Viết Playbook `rolling-update.yml` nhắm `webservers`, `serial: 2`, `max_fail_percentage: 25`, thực hiện một thay đổi có thể quan sát được (ví dụ đổi nội dung file `/healthz` thêm version mới).
2. Chạy `--list-hosts` trước để xác nhận đúng số đợt, đúng host mỗi đợt.
3. Chạy thật, trong lúc chạy dùng một terminal khác liên tục `curl` qua Load Balancer — quan sát và ghi log để xác nhận **không có lúc nào toàn bộ backend đều lỗi cùng lúc**.
4. Thử nghiệm `serial` dạng progressive `[1, 2, "100%"]`, chạy lại, so sánh log số đợt với cách chạy `serial: 2` cố định.

**Kết quả cần có:** 2 Playbook rolling update (cố định và progressive), log `curl` liên tục trong lúc rolling update chứng minh không gián đoạn hoàn toàn.

## Lab 3 — Zero Downtime đầy đủ (gỡ khỏi pool → cập nhật → health check → đưa lại)

**Mục tiêu:** triển khai đúng chuỗi 6 bước ở [[03-Architecture|03 - Architecture]] Sơ đồ 3.

**Các bước:**

1. Viết Playbook thực hiện đủ 6 bước cho từng web server: gỡ khỏi HAProxy pool (dùng lệnh điều khiển HAProxy runtime API hoặc socket, `delegate_to` Load Balancer), chờ drain, cập nhật, restart, health check (`until`/`retries`), đưa lại vào pool.
2. Chạy với `serial: 1` để dễ quan sát từng bước cho từng host.
3. Trong lúc chạy, `curl` liên tục qua Load Balancer, xác nhận traffic **không bao giờ** được gửi tới node đang trong giai đoạn cập nhật (kiểm tra bằng cách mỗi web server trả về hostname/version riêng trong response).
4. Cố tình làm health check thất bại trên 1 node (ví dụ sửa endpoint trả về HTTP 500 tạm thời), xác nhận node đó **không** được đưa lại vào pool, log ghi rõ lý do.

**Kết quả cần có:** Playbook zero downtime đầy đủ, log `curl` liên tục xác nhận zero downtime, log minh họa trường hợp health check thất bại được xử lý đúng.

## Lab 4 — Backup Database trước triển khai

**Mục tiêu:** tích hợp backup tự động vào quy trình, không phải bước thủ công.

**Các bước:**

1. Viết Task backup database (dùng `pg_dump`, `mysqldump`, hoặc backup file cấu hình nếu chưa có database thật) với tên file có timestamp từ Facts (`ansible_date_time`).
2. Thêm bước xác nhận file backup tồn tại và không rỗng (dùng `stat` + `failed_when` như ví dụ ở [[02-Theory|02 - Theory]] mục 6).
3. Đặt Task backup này **trước** mọi Task thay đổi trên Database tier trong Playbook chính, dùng `block`/`rescue` — nếu backup thất bại, `rescue` phải dừng và **không** cho phép các bước thay đổi tiếp theo chạy.
4. Kiểm chứng: cố tình làm backup thất bại (ví dụ đường dẫn không tồn tại), xác nhận Playbook dừng đúng trước khi chạm tới bước thay đổi database.

**Kết quả cần có:** Playbook backup tích hợp, log minh họa trường hợp backup thất bại chặn đúng các bước sau.

## Lab 5 — Monitoring Deployment cơ bản

**Mục tiêu:** Playbook tự báo cáo trạng thái triển khai.

**Các bước:**

1. Viết Task cuối Playbook chính, dùng `run_once: true` và `delegate_to: localhost`, gửi một request (có thể giả lập bằng ghi log ra file JSON cục bộ nếu chưa có hệ thống monitoring thật) ghi lại: thời gian, phiên bản triển khai, danh sách host đã cập nhật thành công.
2. Chạy toàn bộ pipeline (backup → rolling update → zero downtime → thông báo) như một Playbook thống nhất.
3. Xác nhận log/thông báo cuối cùng phản ánh đúng thực tế đã triển khai (không bị gọi nhầm N lần cho N host nhờ `run_once`).

**Kết quả cần có:** Playbook tổng hợp toàn bộ pipeline module này, file/log thông báo kết quả triển khai cuối cùng.

## Checklist hoàn thành module

- [ ] Dựng được kiến trúc 3 tầng, Load Balancer phân phối đúng tới Web tier.
- [ ] Thực hiện Rolling Update bằng `serial`, kiểm chứng bằng log `curl` liên tục.
- [ ] Triển khai đầy đủ chuỗi 6 bước Zero Downtime, xử lý đúng trường hợp health check thất bại.
- [ ] Tích hợp Backup có kiểm tra kết quả, chặn đúng các bước sau nếu backup thất bại.
- [ ] Có Playbook thông báo kết quả triển khai dùng `run_once`.
