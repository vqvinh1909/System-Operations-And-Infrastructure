---
title: "08 - Summary"
module: 0
tags: [ansible, sysops-infra, module-00, summary]
---

# 08 — Tóm tắt & Cầu nối

## Những gì đã học

- **Traditional vs IaC:** quản trị thủ công không lặp lại được, gây configuration drift, không có audit trail, không scale — IaC giải quyết cả 4 vấn đề bằng cách quản lý hạ tầng qua code trong Git.
- **Infrastructure Lifecycle:** Plan → Write/Provision → Review → Apply → Test → Monitor → Update, là một vòng lặp liên tục, không phải quy trình một lần.
- **Declarative vs Imperative:** Ansible declarative ở cấp task (module tự quyết định hành động), imperative ở cấp playbook (thứ tự thực thi do bạn kiểm soát).
- **Agent-based vs Agentless:** Ansible chọn agentless — chỉ cần SSH + Python có sẵn, không cài thêm phần mềm thường trực, tận dụng lại hạ tầng SSH đã hardening từ khóa Linux Sysadmin.
- **Idempotency:** nguyên lý cốt lõi xuyên suốt toàn khóa — chạy lại an toàn, dùng được cho cả provisioning lẫn compliance check.
- **Mutable vs Immutable Infrastructure:** Ansible phù hợp mô hình Mutable (sửa server đang sống); container/Kubernetes (các khóa sau) thiên về Immutable.
- **GitOps (nhập môn):** Git là nguồn chân lý duy nhất, mọi thay đổi hạ tầng đi qua commit — review — merge.
- **Enterprise IaC Architecture:** Git repo → CI/CD → Control Node → Managed Nodes, có lớp review và secrets management — bức tranh này sẽ được hiện thực hóa dần từ Module 02 đến Module 07.

## Tự kiểm tra nhanh

Trước khi sang Module 01, bạn nên tự trả lời được (không cần xem lại bài) các câu hỏi ở [[07-Interview|07 - Interview]] phần Junior. Nếu còn mơ hồ về Idempotency hoặc Agentless, nên đọc lại [[02-Theory|02 - Theory]] mục 4-5 trước khi tiếp tục — đây là hai khái niệm nền tảng nhất, sẽ xuất hiện lại ở **mọi** module tiếp theo.

## Cầu nối sang Module 01

Module 01 — Linux for Automation Review — sẽ **không dạy lại Linux từ đầu** (bạn đã học kỹ ở khóa Linux Sysadmin). Thay vào đó, module này ôn nhanh và **định hướng lại góc nhìn** những kỹ năng Linux bạn đã có (SSH, SSH key, sudo, YAML, Jinja2 cơ bản) dưới lăng kính "cái này sẽ được Ansible dùng như thế nào" — chuẩn bị trực tiếp cho Module 02, nơi bạn cấu hình Ansible control node đầu tiên và chạy ad-hoc command thật.

**Tiếp theo:** [[../01-Linux-for-Automation-Review/README|Module 01 — Linux for Automation Review]]
