---
title: "System Operations & Infrastructure — Master Table of Contents"
tags: [ansible, docker, sysops-infra, toc, curriculum]
status: draft
created: 2026-07-12
course: SysOps-DevOps/System-Operations-Infrastructure
---

# System Operations & Infrastructure — Master Table of Contents (TOC)

> [!info] Vị trí trong lộ trình
> Khóa thứ 4 trong lộ trình 5 khóa: 01-DevOps Automation CI/CD → 02-SysOps Mastery → **03-Linux System Administrator (đã xong, 29 module)** → **04-System Operations & Infrastructure (khóa này)** → 05-Kubernetes for SysOps.
> Khóa này là cầu nối bắt buộc giữa Linux System Administrator và Kubernetes: học viên đã vững Linux, giờ học cách **tự động hóa hạ tầng** (Ansible) và **đóng gói/vận hành ứng dụng** (Docker) — hai kỹ năng nền tảng để hiểu Kubernetes sau này thay vì học nhảy cóc.

## Mục tiêu đầu ra của toàn bộ khóa học

Sau khi hoàn thành, người học có thể:

- Tự động hóa cài đặt, cấu hình server hàng loạt bằng Ansible (Playbooks, Roles, Vault).
- Đóng gói, chạy, và vận hành ứng dụng bằng Docker (image, network, volume, Compose).
- Triển khai một stack nhiều dịch vụ (web + db + cache + queue) bằng Docker Compose.
- Vận hành ngày-2 (day-2 operations): logging, monitoring, backup, bảo mật container.
- Tự tin bước vào khóa Kubernetes for SysOps mà không thiếu kiến thức nền.

## Nguyên tắc thiết kế

- Trình tự bắt buộc: **Ansible (Part I) → Docker (Part II) → Operations (Part III)**, không nhảy cóc.
- Mỗi khái niệm giải thích "Tại sao" trước "Làm như thế nào", có phần Internal Working ở chỗ quan trọng (namespaces/cgroups, VFS layer, Ansible execution model...).
- Mọi thông tin phiên bản/best practice đã được WebSearch xác nhận tính đến 2026-07-12, ghi trong Self-Review từng module.
- Sơ đồ kiến trúc dùng ASCII (không Mermaid), đo bằng script trước khi đưa vào tài liệu.
- Module 20 (capstone) tổng hợp toàn bộ khóa thành một dự án triển khai hạ tầng doanh nghiệp thật.

---

## Tổng quan cấu trúc (21 module, 3 Part)

| Part | Phạm vi | Số module |
|---|---|---|
| I — Infrastructure as Code (Ansible) | Tự động hóa cấu hình hạ tầng | 00–07 (8 module) |
| II — Container Fundamentals (Docker) | Đóng gói và vận hành ứng dụng | 08–15 (8 module) |
| III — Operations | Registry, giám sát, bảo mật, troubleshooting, capstone | 16–20 (5 module) |

---

## PART I — INFRASTRUCTURE AS CODE (ANSIBLE)

### Module 00 — Infrastructure Automation Fundamentals
Traditional Infrastructure vs IaC, Infrastructure Lifecycle, Declarative vs Imperative, Agent vs Agentless, Idempotency, Immutable Infrastructure, GitOps Introduction, Enterprise IaC Architecture.
**Prerequisites:** [[../03-Linux System Administrator/28-Enterprise-Capstone-Project/README|Linux Sysadmin - Module 28]]

### Module 01 — Linux for Automation Review
SSH, SSH Key, SSH Config, Sudo, Inventory Planning, YAML Basics, Jinja2 Basics — ôn nhanh khía cạnh Linux liên quan trực tiếp tới automation.
**Prerequisites:** [[00-Infrastructure-Automation-Fundamentals/README|Module 00]]

### Module 02 — Ansible Fundamentals
Architecture, Control/Managed Node, Inventory (Static, Dynamic, Plugin, Cloud), Variables, Facts, Modules, Ad-hoc Commands, Configuration.
**Prerequisites:** Module 01

### Module 03 — Playbooks
YAML, Tasks, Variables, Loops, Conditionals, Blocks, Handlers, Tags, Includes, Imports, Error Handling.
**Prerequisites:** Module 02

### Module 04 — Templates
Jinja2, Templates, Filters, Lookup, File Module, Copy, Fetch, Synchronize.
**Prerequisites:** Module 03

### Module 05 — Variables, Vault, Secrets
Variable Precedence, Ansible Vault, Secrets Management Best Practices.
**Prerequisites:** Module 04

### Module 06 — Roles
Directory Structure, Defaults, Vars, Meta, Dependencies, Galaxy, Collections, Best Practices.
**Prerequisites:** Module 05

### Module 07 — Enterprise Automation
Multi-tier Deployment, Rolling Update, Zero Downtime, HAProxy, Nginx, Database, Backup, Monitoring Deployment.
**Prerequisites:** Module 06

---

## PART II — CONTAINER FUNDAMENTALS (DOCKER)

### Module 08 — Container Technology
Virtual Machine vs Container, OCI, Namespaces, Cgroups, OverlayFS, Image Layer.
**Prerequisites:** Module 07

### Module 09 — Docker Engine
Docker Architecture (dockerd/containerd/runc/shim), Docker Daemon, CLI, Registry, Image, Container Lifecycle.
**Prerequisites:** Module 08

### Module 10 — Docker Images
Dockerfile, Build, Cache, Multi-stage Build, BuildKit, Best Practices.
**Prerequisites:** Module 09

### Module 11 — Container Runtime
Container Lifecycle, Resource Limit, Restart Policy, Logs, Exec, Inspect, Docker Events, Stats, System df, Prune.
**Prerequisites:** Module 10

### Module 12 — Networking
Bridge, Host, None, Overlay, Macvlan, DNS, Port Mapping.
**Prerequisites:** Module 11

### Module 13 — Storage
Bind Mount, Volume, tmpfs, Backup, Restore.
**Prerequisites:** Module 12

### Module 14 — Docker Compose
Compose Specification, Services, Networks, Volumes, Profiles, Environment, Secrets, Healthcheck.
**Prerequisites:** Module 13

### Module 15 — Multi-container Applications
Web, Database, Redis, RabbitMQ, Nginx Reverse Proxy — ghép thành stack hoàn chỉnh bằng Docker Compose.
**Prerequisites:** Module 14

---

## PART III — OPERATIONS

### Module 16 — Image Registry
Docker Hub, Private Registry, Harbor, Image Security.
**Prerequisites:** Module 15

### Module 17 — Logging & Monitoring
Docker Logs, Loki/Grafana Alloy, Prometheus, cAdvisor, Grafana — stack giám sát container-native, bổ sung cho Zabbix (host-level) đã học trước đó.
**Prerequisites:** Module 16

### Module 18 — Security
Rootless Docker, Capability, Seccomp, AppArmor, Image Scan, Docker Bench for Security.
**Prerequisites:** Module 17

### Module 19 — Troubleshooting
Container Debugging, Network Debugging, Storage Debugging, Performance Analysis — tổng hợp chuyên sâu.
**Prerequisites:** Module 18

### Module 20 — Enterprise Project (Capstone)
5 VM (Control Node, HAProxy, Web01, Web02, Database) → Ansible provisioning → Docker + Compose deploy → Monitoring → Backup → 20 tình huống troubleshooting thực tế.
**Prerequisites:** Module 19
**Tiếp theo:** Kubernetes for SysOps (khóa 05, chưa xây)

---

> [!note] Ghi chú
> TOC này được viết SAU khi nội dung 21 module đã hoàn thành, dùng làm bản đồ điều hướng — khác với TOC khóa 03 (viết trước khi có nội dung, dùng để duyệt phạm vi).
