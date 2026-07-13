---
title: "01 - Introduction"
module: 5
tags: [ansible, sysops-infra, module-05, introduction]
---

# 01 — Giới thiệu

## Bài toán 1: "Sao biến này lại ra giá trị khác tôi nghĩ?"

Bạn khai báo `http_port: 8080` trong `group_vars/webservers.yml`. Đồng nghiệp khác khai báo `http_port: 9090` trong `host_vars/web-01.yml`. Bạn chạy Playbook với `--extra-vars "http_port=7070"`. Web-01 sẽ chạy port nào? Nếu bạn không thuộc lòng thứ tự ưu tiên biến, câu trả lời chỉ là đoán — và đoán sai trong một buổi triển khai production là sự cố thật, không phải bài tập lý thuyết. Đây là lý do phần đầu module này dành để học **thuộc lòng** bảng Variable Precedence.

## Bài toán 2: "Mật khẩu database để ở đâu trong file Ansible commit lên Git?"

Playbook của bạn cần biết mật khẩu database để cấu hình ứng dụng. Bạn **không thể** viết `db_password: "S3cret123"` thẳng vào file YAML rồi `git push` — bất kỳ ai có quyền đọc repo (kể cả trong quá khứ, qua lịch sử Git) đều thấy được mật khẩu. Đây không phải lỗi lý thuyết — là nguyên nhân hàng đầu của các vụ rò rỉ secret thực tế trong ngành (secret bị commit nhầm vào public repo, hoặc private repo nhưng quá nhiều người có quyền đọc). **Ansible Vault** giải quyết bài toán này: mã hóa nội dung ngay trong hệ sinh thái Ansible, commit bản mã hóa lên Git một cách an toàn, chỉ người có password Vault mới giải mã được.

## Vì sao gộp hai chủ đề này vào một module

Variable Precedence và Vault tưởng như hai chủ đề tách biệt, nhưng có liên hệ chặt: **Vault mã hóa chính là một cách đặc biệt để khai báo biến** — file Vault vẫn tuân theo đúng thứ tự ưu tiên như biến thường, chỉ khác là nội dung được mã hóa khi lưu trữ. Hiểu rõ precedence là điều kiện cần để hiểu đúng "biến Vault của tôi có thực sự được dùng hay bị biến khác ghi đè".

## Mục tiêu học tập

Sau module này, bạn thuộc lòng thứ tự ưu tiên biến của Ansible, debug được các tình huống biến ra giá trị bất ngờ, mã hóa/giải mã thành thạo bằng `ansible-vault`, và áp dụng được các nguyên tắc Secrets Management tối thiểu để không bao giờ để lộ plaintext secret trong Git — nền tảng bắt buộc trước khi đóng gói mọi thứ thành Role tái sử dụng ở Module 06.
