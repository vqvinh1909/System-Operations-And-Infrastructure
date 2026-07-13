---
title: "03 - Architecture"
module: 16
tags: [docker, sysops-infra, module-16, registry, harbor, architecture]
---

# 03 — Kiến trúc: Luồng Push/Pull và Nội bộ Harbor

> [!note] Cách đo sơ đồ trong file này
> Mọi khung (box) dưới đây được dựng bằng script Python nội bộ (hàm `box()` tính `inner_width = max(len(dòng)) + padding*2`, dùng đúng `inner_width` đó cho border trên, border dưới, và mọi dòng nội dung) — đảm bảo `len(border trên) == len(border dưới) == len(mọi dòng nội dung)` cho từng khung một cách chắc chắn bằng toán học, không dựa vào đếm tay. Các dòng mũi tên/kết nối giữa hai khung (`|`, `v`, nhãn chú thích) không phải là "khung" nên không áp dụng quy tắc này — chỉ áp dụng cho chính khung chữ nhật.

## Sơ đồ 1: Luồng Push/Pull image qua Private Registry

```
+-----------------+
| Dev Workstation |
| (docker build)  |
+-----------------+
         |
         | docker push (sau khi build xong, gan tag registry.company.local/...)
         v
+---------------------------+
| Private Registry (Harbor) |
| registry.company.local    |
+---------------------------+
         |
         | docker pull (ca CI/CD pipeline va production server deu pull tu day)
         v
+-----------------------+
| CI/CD Pipeline        |
| (Jenkins / GitLab CI) |
+-----------------------+

              -- song song, khong phu thuoc CI/CD Pipeline --

+------------------------+
| Production Servers     |
| docker pull / K8s pull |
+------------------------+
```

Đọc sơ đồ theo đúng thứ tự nghiệp vụ thật:

1. Dev build image trên máy cá nhân, gắn tag trỏ về registry nội bộ (ví dụ `registry.company.local/payment-service:v2.3` — **không phải** `payment-service:v2.3` trần trụi, vì Docker mặc định hiểu tag không có domain là thuộc Docker Hub).
2. `docker push` đẩy image lên Private Registry — đây là điểm bàn giao duy nhất giữa "máy cá nhân" và "hạ tầng chung".
3. Từ đây, CI/CD pipeline và các server production **hoàn toàn độc lập** pull image về theo nhu cầu riêng của từng bên — registry đóng vai trò một nguồn sự thật duy nhất (single source of truth) cho mọi môi trường.

## Sơ đồ 2: Kiến trúc nội bộ Harbor — luồng chính (Portal → Core → Registry → Storage)

```
+-----------------+
| Portal (Web UI) |
+-----------------+
         |
         | HTTPS (nguoi dung thao tac qua trinh duyet)
         v
+-------------------------------+
| Harbor Core                   |
| (auth, RBAC, project quan ly) |
+-------------------------------+
         |
         | luu/doc metadata cua image, user, policy
         v
+------------------------------------+
| Registry                           |
| (Distribution - luu blob/manifest) |
+------------------------------------+
         |
         | doc/ghi blob thuc te (layer, config)
         v
+--------------------------------+
| Storage Backend                |
| (Filesystem / S3 / Azure Blob) |
+--------------------------------+
```

**Harbor Core** là bộ não trung tâm — mọi request (từ Portal hoặc từ `docker push`/`docker pull` trực tiếp) đều đi qua Core trước để xác thực (authentication) và kiểm tra quyền (authorization theo RBAC của project), rồi mới được chuyển tiếp xuống **Registry** (chính là engine Distribution — cùng công nghệ với `registry:2` đã học ở phần lý thuyết) để thực sự đọc/ghi blob xuống **Storage Backend**.

## Sơ đồ 3: Các thành phần phụ trợ của Harbor Core

```
Harbor Core con giao tiep truc tiep voi 3 thanh phan sau
(ve rieng de tranh so do bi roi vi qua nhieu duong noi):

+-----------------------------------+
| PostgreSQL                        |
| (metadata: user, project, policy) |
+-----------------------------------+

+--------------------+
| Redis              |
| (cache, job queue) |
+--------------------+

+-----------------------------+
| Job Service                 |
| (replication, GC, scan job) |
+-----------------------------+
         |
         | kich hoat scan job
         v
+-------------------------+
| Trivy                   |
| (vulnerability scanner) |
+-------------------------+
```

- **PostgreSQL** lưu toàn bộ metadata quan trọng: danh sách user, project, policy phân quyền, cấu hình replication. Đây là lý do backup Harbor **bắt buộc phải backup cả PostgreSQL lẫn Storage Backend** — mất một trong hai là mất khả năng khôi phục đầy đủ (mất PostgreSQL: còn blob nhưng không biết ai sở hữu, quyền gì; mất Storage: còn metadata nhưng không còn image thật).
- **Redis** dùng làm cache và hàng đợi job (job queue) — khi Redis down, Harbor UI thường vẫn load được nhưng các thao tác nền (scan, replication, GC) sẽ bị treo hoặc lỗi.
- **Job Service** là tiến trình chạy nền, thực thi các tác vụ bất đồng bộ: replication (đồng bộ image sang Harbor instance khác), garbage collection (dọn blob không còn manifest nào tham chiếu tới), và kích hoạt **Trivy** để scan image tìm lỗ hổng.

## Internal Working: điều gì xảy ra khi bạn `docker push` vào Harbor

1. Docker client gửi request `PUT` chứa từng blob (layer) lần lượt tới endpoint `/v2/<project>/<repo>/blobs/uploads/...` — đây chính là giao thức OCI Distribution Spec đã học ở [[02-Theory]].
2. Request đầu tiên chạm **Harbor Core**, không chạm thẳng Registry — Core kiểm tra: user này có tài khoản hợp lệ không (authentication), có quyền push vào project này không (authorization theo RBAC).
3. Nếu hợp lệ, Core **proxy** (chuyển tiếp) request xuống Registry — Registry mới thực sự nhận blob và ghi xuống Storage Backend.
4. Sau khi toàn bộ blob upload xong, client gửi manifest — Registry ghi manifest, đồng thời Harbor Core cập nhật metadata (ai push, lúc nào, digest nào) vào PostgreSQL.
5. Nếu project có bật "Automatically scan images on push" (chính sách phổ biến trong doanh nghiệp), Job Service tạo một scan job, giao cho Trivy chạy nền — kết quả scan được ghi lại và hiển thị trên Portal, không chặn lệnh `docker push` (scan chạy bất đồng bộ), nhưng có thể chặn việc **pull/deploy** sau đó nếu policy được cấu hình nghiêm ngặt ("Prevent vulnerable images from running").

> [!warning] Vì sao hiểu đúng bước 2-3 quan trọng khi troubleshoot
> Nếu bạn thấy lỗi push nhưng log của Registry (container `registry` bên trong Harbor) không ghi nhận gì cả, nhiều khả năng request đã bị Harbor Core chặn **trước khi tới Registry** (do lỗi auth/permission) — lúc đó bạn phải xem log của Core, không phải log của Registry. Đây là sai lầm rất phổ biến khi troubleshoot Harbor lần đầu (xem thêm [[06-Troubleshooting]]).
