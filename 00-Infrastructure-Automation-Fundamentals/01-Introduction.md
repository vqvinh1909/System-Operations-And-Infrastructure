---
title: "01 - Introduction"
module: 0
tags: [ansible, sysops-infra, module-00, introduction, iac]
---

# 01 — Giới thiệu

## Bối cảnh: vì sao module này tồn tại

Bạn vừa hoàn thành 29 module Linux System Administrator — biết cài hệ điều hành, cấu hình SSH, quản lý user, systemd, firewall, hardening bảo mật. Bạn có thể tự tay dựng một server từ A đến Z. Câu hỏi tiếp theo mà bất kỳ System Administrator/DevOps nào cũng phải trả lời là: **"Nếu phải làm lại việc đó 50 lần, trên 50 server, trong 2 giờ, bạn làm thế nào?"**

Đây chính xác là tình huống xảy ra hàng ngày trong doanh nghiệp:
- Một công ty thương mại điện tử cần scale từ 5 lên 30 web server trước đợt sale Black Friday — trong vài giờ, không phải vài ngày.
- Một đội bảo mật phát hiện lỗ hổng OpenSSH nghiêm trọng, cần patch **đồng loạt** 200 server trong đêm.
- Một audit tuân thủ (compliance) yêu cầu chứng minh **mọi** server đều có cùng một cấu hình firewall, cùng version, cùng chính sách password — không có ngoại lệ.

Không đội vận hành nào làm nổi việc này bằng cách SSH thủ công vào từng máy. Đây là lý do ngành sinh ra **Infrastructure as Code (IaC)** — và module này xây nền tư duy trước khi bạn học công cụ (Ansible) ở các module sau.

## Vì sao học IaC ngay sau Linux, trước Docker/Kubernetes?

Thứ tự này không ngẫu nhiên:
- Bạn phải **biết cấu hình một server bằng tay trước** (đã học ở khóa Linux Sysadmin) thì mới hiểu được Ansible đang tự động hóa **cái gì** — nếu học Ansible trước, bạn sẽ gõ theo mẫu mà không hiểu module `systemd` hay `user` bên dưới thực chất đang chạy lệnh gì.
- IaC (Ansible) là kỹ năng **quản lý hạ tầng đang tồn tại** (server vật lý, VM) — còn Container (Docker, module tiếp theo Part II) và Kubernetes (khóa 05) là **đơn vị triển khai ứng dụng mới**. Trong thực tế doanh nghiệp, cả hai luôn song hành: dùng Ansible để dựng và vá hệ điều hành nền (kể cả node Kubernetes), dùng Docker/Kubernetes để chạy ứng dụng bên trên.

## Mục tiêu học tập của module

Sau module này, bạn có thể trả lời rành mạch (kể cả trong phỏng vấn) các câu hỏi:
- Traditional Infrastructure Management khác Infrastructure as Code ở điểm nào, chi phí ẩn của cách làm cũ là gì?
- Declarative và Imperative khác nhau ra sao, Ansible thiên về hướng nào?
- Agent-based và Agentless khác nhau ra sao, đánh đổi (trade-off) của từng mô hình?
- Idempotency là gì, tại sao nó là **nguyên lý sống còn** của mọi công cụ automation?
- Mutable và Immutable Infrastructure khác nhau ra sao?
- GitOps là gì ở mức khái niệm nhập môn?
- Một kiến trúc IaC cấp doanh nghiệp gồm những thành phần nào?

## Cách tiếp cận của module

Module này **không có lệnh Ansible nào** — đây là lựa chọn có chủ đích. Idempotency, Declarative, Agentless là những khái niệm bạn sẽ gặp lại ở **mọi** module sau (02 đến 07); nếu hiểu sai ngay từ đầu, càng học sâu càng rối. Hãy đọc chậm phần [[02-Theory|02 - Theory]], đối chiếu với sơ đồ ở [[03-Architecture|03 - Architecture]], rồi tự tay làm lab ở [[05-Labs|05 - Labs]] (dùng shell script thường, chưa cần Ansible) để cảm nhận thực tế "thế nào là idempotent" trước khi bước sang Module 01.
