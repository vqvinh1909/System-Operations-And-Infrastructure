---
title: "08 - Summary"
module: 16
tags: [docker, sysops-infra, module-16, registry, harbor, summary]
---

# 08 — Tóm tắt Module 16

## Những gì bạn đã học

- **Registry** là dịch vụ HTTP lưu và phân phối image theo chuẩn OCI Distribution Spec, tách biệt hai loại dữ liệu: blob (layer, config) và manifest (mô tả cách ghép blob thành một image).
- **Docker Hub** đúng vai trò là nguồn base image công khai, không phải nơi lưu image nội bộ doanh nghiệp — vì lý do bảo mật, RBAC, compliance, và chi phí băng thông.
- **`registry:2`** là cách nhanh nhất để có một registry hoạt động, nhưng thiếu UI, RBAC chi tiết, scanning tích hợp — phù hợp lab/CI nhỏ, không phù hợp làm registry chính doanh nghiệp.
- **Harbor** là registry doanh nghiệp phổ biến nhất hiện nay (CNCF Graduated), bọc lớp quản trị (Portal, Core, RBAC theo project, Job Service, Trivy, replication, robot account) lên trên chính engine Distribution.
- **Image scanning** đối chiếu package trong image với vulnerability database để phát hiện CVE — cần chạy định kỳ chứ không chỉ một lần lúc push, vì database luôn cập nhật thêm lỗ hổng mới.
- Vận hành thực tế: TLS/auth bắt buộc, Garbage Collection định kỳ để không phình dung lượng, robot account cho CI/CD thay vì tài khoản cá nhân.

## Bảng đối chiếu nhanh

| Tiêu chí | Docker Hub | `registry:2` | Harbor |
|---|---|---|---|
| Vị trí | Public internet | Tự host | Tự host |
| UI quản trị | Có (web) | Không | Có (Portal) |
| RBAC theo project | Giới hạn | Không | Có |
| Image scanning tích hợp | Docker Scout (riêng) | Không | Trivy (tích hợp) |
| Replication | Không | Không | Có |
| Phù hợp | Base image công khai | Lab/CI nhỏ | Registry chính doanh nghiệp |

## Cầu nối sang Module 17

Bạn đã có nơi lưu trữ image an toàn. Nhưng registry chỉ giải quyết bài toán "image nằm ở đâu" — nó không cho bạn biết **container đang chạy image đó hoạt động ra sao**: có đang log lỗi liên tục không, có đang ăn hết CPU/RAM không, có đang bị crash-loop không. Đó chính là nội dung [[../17-Logging-Monitoring/README|Module 17 — Logging & Monitoring]], nơi bạn học cách quan sát (observability) toàn bộ container đang chạy trong hạ tầng, dùng đúng bộ công cụ container-native (Loki, Prometheus, cAdvisor, Grafana) thay vì công cụ giám sát host-level truyền thống bạn đã biết.
