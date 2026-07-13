---
title: "01 - Introduction"
module: 20
tags: [docker, ansible, sysops-infra, module-20, capstone]
---

# 01 — Giới thiệu Module: Enterprise Project

## Bối cảnh dự án

Bạn được giao vai trò sysadmin/DevOps duy nhất cho một công ty vừa quyết định đưa một ứng dụng web nội bộ (kèm database, cache, message queue) lên một hạ tầng riêng thay vì tiếp tục chạy tạm trên một server cá nhân của dev. Yêu cầu từ quản lý rất ngắn gọn: **"Dựng cho công ty một hạ tầng chạy được, có khả năng chịu tải cơ bản (không phụ thuộc một server duy nhất), có giám sát để biết khi nào có sự cố, và có backup để không mất dữ liệu nếu có gì hỏng."**

Đây chính là bản tóm tắt của gần như mọi dự án hạ tầng thực tế mà một sysadmin/DevOps cấp junior-to-mid sẽ được giao trong công việc đầu tiên. Module 20 mô phỏng lại đúng bối cảnh đó, ở quy mô đủ nhỏ để bạn tự làm một mình trong môi trường lab, nhưng đủ đầy đủ để phản ánh đúng những quyết định kiến trúc và vận hành bạn sẽ gặp lại nhiều lần trong sự nghiệp.

## Vì sao module này là bài kiểm tra tổng hợp, không phải bài học mới

Suốt 19 module trước, bạn học từng mảnh ghép riêng lẻ: Ansible (Part I) dạy bạn tự động hóa cấu hình, Docker (Part II) dạy bạn đóng gói và chạy ứng dụng, Part III (Registry, Logging & Monitoring, Security, Troubleshooting) dạy bạn vận hành những container đó ở quy mô doanh nghiệp. Nhưng trong công việc thật, không ai giao cho bạn "bài tập Ansible" tách biệt "bài tập Docker" — mọi dự án thật đều là **sự kết hợp** của tất cả các mảnh ghép đó cùng lúc.

Module 20 không dạy khái niệm mới. Nó yêu cầu bạn **tự quyết định** cách kết hợp mọi thứ đã học: Ansible role nào cần viết, service Docker nào chạy trên node nào, network Docker Compose thiết kế ra sao để Web01/Web02 gọi được Database nhưng vẫn cô lập hợp lý, dashboard Grafana nào thật sự cần thiết cho hạ tầng cụ thể này — không có một "đáp án đúng duy nhất" như các module lý thuyết trước, chỉ có các quyết định kiến trúc hợp lý hay không hợp lý dựa trên nguyên tắc đã học.

## Mục tiêu học tập

Sau module này, bạn sẽ:

1. Có kinh nghiệm thực tế dựng một hạ tầng nhiều VM từ đầu tới cuối, không chỉ làm theo từng bài lab tách biệt.
2. Tự tin viết một bộ Ansible playbook/role hoàn chỉnh cho một dự án thật, không chỉ chép lại ví dụ mẫu.
3. Tự tin thiết kế kiến trúc Docker Compose nhiều service với network/volume hợp lý cho một ứng dụng nhiều tầng (web, cache, queue, database).
4. Tự tin chẩn đoán sự cố trên một hạ tầng do chính mình dựng, không có ai chỉ sẵn "lỗi nằm ở đây".
5. Có một dự án hoàn chỉnh để đưa vào portfolio cá nhân, kể lại được trong phỏng vấn xin việc như một dự án thật đã từng làm.

## Kiến thức nền cần có trước

- Đã hoàn thành toàn bộ Module 00 đến Module 19 của khóa 04. Module này không dạy lại bất kỳ khái niệm nào — nếu bạn thấy khó hiểu một thuật ngữ ở [[05-Labs]], quay lại đúng module tương ứng đã học trước đó thay vì tìm lời giải thích lại ở đây.
- Có sẵn môi trường có thể dựng tối thiểu 5 máy ảo (VirtualBox/VMware/cloud VM nhỏ đều được) — xem yêu cầu tài nguyên cụ thể ở [[05-Labs]].
