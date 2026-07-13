---
title: "08 - Summary"
module: 20
tags: [docker, ansible, sysops-infra, module-20, capstone, summary]
---

# 08 — Tổng kết toàn khóa: System Operations & Infrastructure

## Bạn đã đi qua một hành trình 21 module

Nhìn lại từ điểm xuất phát: bạn bắt đầu khóa này với kiến thức Linux Sysadmin vững chắc nhưng chưa biết tự động hóa hạ tầng ở quy mô nhiều máy, chưa biết đóng gói và vận hành ứng dụng bằng container. Sau 21 module, bạn đã tự tay dựng được một hạ tầng doanh nghiệp hoàn chỉnh — triển khai tự động, giám sát được, bảo mật theo nhiều lớp, có khả năng chẩn đoán sự cố và khôi phục sau thảm họa.

**Part I — Automation Foundations (Module 00-07):** bắt đầu từ ôn tập Linux phục vụ automation, qua Ansible Fundamentals, Playbooks, Templates, Variables/Vault/Secrets, Roles, tới Enterprise Automation — bạn học cách biến "cấu hình một server" từ thao tác thủ công lặp đi lặp lại thành mã nguồn có thể tái sử dụng, review, và version control (Infrastructure as Code).

**Part II — Container Technology (Module 08-15):** từ khái niệm nền tảng của container (namespace, cgroup), qua Docker Engine, Docker Images, Container Runtime, Networking, Storage, tới Docker Compose và Multi-container Applications — bạn học cách đóng gói ứng dụng thành đơn vị triển khai nhất quán, độc lập với môi trường bên dưới.

**Part III — Operations (Module 16-20, phần bạn vừa hoàn thành):** từ Image Registry (lưu trữ và phân phối image ở quy mô tổ chức), Logging & Monitoring (quan sát container-native bằng Loki/Prometheus/Grafana), Security (thu hẹp bề mặt tấn công bằng Rootless/Capabilities/Seccomp/AppArmor/Image Scanning), Troubleshooting (quy trình chẩn đoán phân tầng có hệ thống), tới Enterprise Project (đồ án tổng hợp toàn bộ kiến thức trên một hạ tầng 5 VM thật) — bạn học cách **vận hành** những gì Part I và Part II đã xây dựng, ở mức độ trưởng thành mà một tổ chức thật yêu cầu.

## Bảng tổng hợp năng lực đã đạt được

| Lĩnh vực | Trước khóa học | Sau khóa học |
|---|---|---|
| Cấu hình hạ tầng | Thủ công từng server | Tự động hóa bằng Ansible, idempotent, có version control |
| Đóng gói ứng dụng | Cài đặt trực tiếp trên host | Container hóa bằng Docker, nhất quán qua mọi môi trường |
| Phân phối image | Không có quy trình chuẩn | Registry riêng (Harbor), có scanning, có RBAC |
| Giám sát | Chỉ ở mức host (Zabbix) | Thêm tầng container-native (Loki/Prometheus/Grafana) |
| Bảo mật | Hardening ở mức host | Thêm nhiều lớp ở mức container (rootless, capability, seccomp, AppArmor) |
| Xử lý sự cố | Thử theo kinh nghiệm rời rạc | Quy trình phân tầng có hệ thống, tái sử dụng được |
| Dự án thực tế | Chưa có | Một hạ tầng 5 VM hoàn chỉnh, có thể đưa vào portfolio |

## Điều quan trọng nhất cần nhớ

Công cụ sẽ thay đổi — bạn đã tự chứng kiến điều này ngay trong khóa học (Promtail hết vòng đời, được thay bằng Grafana Alloy chỉ trong quá trình bạn học). Nhưng **nguyên tắc** đằng sau các công cụ đó gần như không đổi: tách biệt cấu hình khỏi mã nguồn, tự động hóa để loại bỏ sai sót thủ công, thu hẹp quyền hạn tới mức tối thiểu cần thiết, luôn có kế hoạch cho khi mọi thứ thất bại (backup, health check, quy trình chẩn đoán). Nắm vững nguyên tắc, công cụ mới nào xuất hiện trong sự nghiệp sau này bạn cũng sẽ học được nhanh.

## Hướng đi tiếp theo

Kiến trúc bạn vừa dựng ở Module 20 — nhiều VM, mỗi VM chạy Docker Compose riêng, HAProxy làm load balancer thủ công — là cách một tổ chức vận hành container ở quy mô vừa và nhỏ. Khi số lượng service và node tăng lên, việc tự quản lý thủ công từng VM, tự viết cấu hình HAProxy, tự đảm bảo container được restart đúng node khi một VM chết, bắt đầu trở nên quá tải cho một team vận hành — đây chính là bài toán mà Kubernetes được sinh ra để giải quyết: tự động hóa việc điều phối (orchestration) container ở quy mô lớn, tự động phân phối tải, tự động khôi phục khi có node chết, quản lý cấu hình và bí mật (secrets) ở quy mô cluster.

Khóa tiếp theo trong lộ trình của bạn, "Kubernetes for SysOps", sẽ xây dựng trực tiếp trên nền tảng bạn vừa hoàn thành ở khóa này — mọi khái niệm cốt lõi bạn đã học (container, network, volume, log driver, capability, health check, load balancing) đều xuất hiện lại trong Kubernetes dưới hình thức trưởng thành và tự động hóa hơn. Bạn không bắt đầu từ số 0 — bạn mang theo toàn bộ nền tảng vững chắc từ 21 module vừa hoàn thành.
