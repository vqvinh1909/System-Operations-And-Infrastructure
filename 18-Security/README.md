---
title: "Module 18 - Security"
tags: [docker, sysops-infra, module-18, security, rootless, seccomp, apparmor]
module: 18
part: "III - Operations"
difficulty: Advanced
status: draft
created: 2026-07-12
prerequisites: ["[[../17-Logging-Monitoring/README|Module 17]]"]
next: "[[../19-Troubleshooting/README|Module 19]]"
---

# Module 18 — Security

> [!info] Vị trí trong giáo trình
> Part III — Operations · Module 18/21 của khóa 04 (System Operations & Infrastructure) · Nối tiếp Module 17 (Logging & Monitoring) — bạn đã biết quan sát container đang chạy, giờ học cách **thu hẹp bề mặt tấn công** của chính container đó trước khi có sự cố bảo mật xảy ra, thay vì chỉ phát hiện sau khi bị tấn công.

Module này liên hệ trực tiếp tới phần **Security Hardening** bạn đã học ở khóa Linux Sysadmin (SELinux/AppArmor mức host, principle of least privilege, giảm quyền root) — áp dụng lại đúng những nguyên tắc đó, nhưng ở phạm vi container thay vì toàn bộ hệ điều hành.

## Mục lục module

| File | Nội dung |
|---|---|
| [[01-Introduction]] | Vì sao container mặc định không an toàn như bạn nghĩ, mục tiêu học tập |
| [[02-Theory]] | Rootless Docker, Capabilities, Seccomp, AppArmor, Image Scanning |
| [[03-Architecture]] | Sơ đồ rootful vs rootless, mô hình phòng thủ theo lớp (defense in depth) |
| [[04-Commands]] | Lệnh cài rootless, `--cap-drop`/`--cap-add`, seccomp profile, Docker Bench |
| [[05-Labs]] | Lab rootless mode, lab capability, lab seccomp/AppArmor, lab scan + Bench |
| [[06-Troubleshooting]] | Lỗi permission khi rootless, container bị seccomp chặn, AppArmor deny |
| [[07-Interview]] | Câu hỏi phỏng vấn về container security |
| [[08-Summary]] | Tóm tắt, cầu nối sang Module 19 (Troubleshooting) |

## Learning Objectives (tổng)

Sau khi hoàn thành module này, người học phải:

1. Giải thích được vì sao Docker rootful (daemon chạy bằng root) là một rủi ro kiến trúc, và Rootless Docker giảm thiểu rủi ro đó như thế nào.
2. Áp dụng được nguyên tắc "principle of least privilege" ở mức container bằng `--cap-drop`/`--cap-add`, hiểu đúng ý nghĩa của từng capability phổ biến.
3. Giải thích được Seccomp và AppArmor bổ sung cho nhau như thế nào (chặn syscall vs chặn truy cập file/tài nguyên), liên hệ được với kiến thức AppArmor/SELinux mức host đã học trước đó.
4. Tự chạy được image scanning (dò lỗ hổng CVE) và Docker Bench for Security để đánh giá một hệ thống Docker theo CIS Benchmark.
5. Thiết kế được một checklist hardening Docker tối thiểu áp dụng được ngay cho môi trường thật.

## Self-Review

> [!note] Self-Review
> - **Technical Accuracy:** Đã WebSearch xác nhận (2026-07-12): Rootless Docker được coi là sẵn sàng cho production ở quy mô single-node/multi-container năm 2026, xu hướng ngành (Docker, Podman, containerd, Kubernetes) đều đầu tư kiến trúc rootless-first; với Kubernetes production quy mô lớn, rootless vẫn đang trong giai đoạn hoàn thiện chứ chưa phải mặc định. Docker Bench for Security (`docker/docker-bench-security`) xác nhận **vẫn được duy trì trên GitHub**, hỗ trợ CIS Docker Benchmark v1.6.0, tuy nhiên image dựng sẵn trên Docker Hub bị ghi nhận là lỗi thời — khuyến nghị clone repo và tự build thay vì pull image có sẵn. Docker mặc định áp dụng seccomp profile chặn khoảng 44 syscall và AppArmor profile `docker-default` — khuyến nghị 2026 là giữ nguyên cấu hình mặc định (không tắt), chỉ tùy biến khi có bằng chứng rõ ràng cần thiết. Không bịa số liệu benchmark nào không kiểm chứng được.
> - **Completeness:** Đã bao phủ Rootless Docker, Capability, Seccomp, AppArmor (có liên hệ Module Security Hardening ở Linux Sysadmin), Image Scan, Docker Bench for Security theo đúng phạm vi đề ra.
> - **ASCII Diagram:** Mọi sơ đồ trong [[03-Architecture]] đã được dựng bằng script Python (hàm `box()` đảm bảo `len()` border trên = border dưới = mọi dòng nội dung cho từng khung) và chạy script `verify_boxes.py` xác nhận "ALL BOXES OK" trước khi đưa vào file.

## Cấu trúc thư mục

```
18-Security/
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
