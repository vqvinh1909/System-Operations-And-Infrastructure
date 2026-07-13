---
title: "Module 16 - Image Registry"
tags: [docker, sysops-infra, module-16, registry, harbor]
module: 16
part: "III - Operations"
difficulty: Advanced
status: draft
created: 2026-07-12
prerequisites: ["[[../15-Multi-container-Applications/README|Module 15]]"]
next: "[[../17-Logging-Monitoring/README|Module 17]]"
---

# Module 16 — Image Registry

> [!info] Vị trí trong giáo trình
> Part III — Operations · Module 16/21 của khóa 04 (System Operations & Infrastructure) · Mở đầu Part III, tiếp nối trực tiếp từ Part II (Docker) — bạn đã biết build và chạy container, module này dạy bạn **lưu trữ và phân phối image đó cho cả một tổ chức**, thay vì chỉ chạy trên máy cá nhân.

Đây là module đầu tiên của Part III — Operations, nơi khóa học chuyển trọng tâm từ "biết dùng Docker" sang "vận hành Docker ở quy mô doanh nghiệp". Image registry là mảnh ghép bắt buộc phải có trước khi nói tới CI/CD thật, vì mọi pipeline build-test-deploy đều cần một nơi trung tâm để lưu và phân phối image đã build.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Vì sao doanh nghiệp cần registry riêng, mục tiêu học tập |
| [[02-Theory]] | Registry là gì, OCI Distribution Spec, Docker Hub vs Private Registry vs Harbor |
| [[03-Architecture]] | Sơ đồ luồng push/pull, kiến trúc nội bộ Harbor |
| [[04-Commands]] | Lệnh chạy `registry:2`, cấu hình `daemon.json`, lệnh Harbor CLI/UI cơ bản |
| [[05-Labs]] | Lab dựng private registry, lab cài Harbor, lab image scanning |
| [[06-Troubleshooting]] | Lỗi push/pull, lỗi TLS/certificate, lỗi storage đầy |
| [[07-Interview]] | Câu hỏi phỏng vấn về registry, Harbor, image security |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 17 (Logging & Monitoring) |

## Learning Objectives (tổng)

Sau khi hoàn thành module này, người học phải:

1. Giải thích được vì sao một tổ chức không nên dùng Docker Hub công khai để lưu image nội bộ chứa mã nguồn/cấu hình nhạy cảm của công ty.
2. Tự dựng được một private registry đơn giản bằng image `registry:2`, và hiểu registry này thiếu những gì so với một registry doanh nghiệp thật.
3. Giải thích được Harbor là gì, vì sao nó là lựa chọn phổ biến trong doanh nghiệp, và các thành phần nội bộ của nó hoạt động cùng nhau như thế nào.
4. Trình bày được khái niệm image scanning, vulnerability database, và vì sao "scan image trước khi deploy" là một bước bắt buộc trong pipeline bảo mật hiện đại.

## Self-Review

> [!note] Self-Review
> - **Technical Accuracy:** Đã WebSearch xác nhận (2026-07-12): Harbor là CNCF graduated project (từ 06/2020), vẫn đang ở nhánh 2.x (bản 2.15.x là bản mới gần nhất được ghi nhận), là registry doanh nghiệp phổ biến nhất hiện tại. Docker Scout đã thay thế `docker scan` (cũ, dựa trên Snyk) làm công cụ scan tích hợp sẵn của Docker. Không bịa số liệu benchmark nào không kiểm chứng được.
> - **Completeness:** Đã bao phủ Docker Hub, Private Registry (`registry:2`), Harbor (kiến trúc + thành phần), khái niệm image scanning và vulnerability database, theo đúng phạm vi đề ra.
> - **ASCII Diagram:** Mọi sơ đồ trong [[03-Architecture]] đã được dựng và đo bằng script Python (`len()` so khớp border trên/dưới/nội dung từng khung) trước khi đưa vào file — xem chi tiết cách đo trong phần đầu file đó.

## Cấu trúc thư mục

```
16-Image-Registry/
├── README.md
├── 01-Introduction.md
├── 02-Theory.md
├── 03-Architecture.md
├── 04-Commands.md
├── 05-Labs.md
├── 06-Troubleshooting.md
├── 07-Interview.md
├── 08-Summary.md
├── images/
├── labs/
└── scripts/
```
