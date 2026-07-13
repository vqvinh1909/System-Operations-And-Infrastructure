---
title: "05 - Labs (Nội dung Đồ án)"
module: 20
tags: [docker, ansible, sysops-infra, module-20, capstone, project]
---

# 05 — Nội dung Đồ án: Dựng hạ tầng doanh nghiệp 5 VM

> [!warning] File này không chứa câu hỏi
> Đây là bản mô tả dự án (project spec) — chỉ trình bày kiến trúc, yêu cầu kỹ thuật, các bước triển khai và tiêu chí hoàn thành. Không có câu hỏi lý thuyết hay quiz ở đây. Câu hỏi phỏng vấn tổng hợp toàn khóa nằm riêng ở [[07-Interview]].

## Mục tiêu đồ án

Dựng một hạ tầng 5 VM hoàn chỉnh cho một ứng dụng web 3 tầng, triển khai hoàn toàn tự động bằng Ansible, các dịch vụ chạy trong Docker, có giám sát tập trung và có chiến lược backup rõ ràng — đúng theo kiến trúc đã trình bày ở [[03-Architecture]].

## Yêu cầu tài nguyên môi trường lab

- Tối thiểu 5 máy ảo (VirtualBox, VMware, hoặc VPS/cloud instance nhỏ đều chấp nhận được), mỗi máy tối thiểu 1 vCPU / 1GB RAM cho các node Web/HAProxy/Control Node, riêng node Database nên có tối thiểu 2GB RAM để chạy đồng thời MySQL, Redis, RabbitMQ.
- Cả 5 VM cùng nằm trong một mạng nội bộ (private network/host-only network) để giao tiếp trực tiếp bằng IP tĩnh, theo đúng bảng IP tham khảo ở [[03-Architecture]].
- Hệ điều hành: dùng cùng distro Linux đã quen thuộc từ khóa Linux Sysadmin, cài đặt Docker Engine trên 4 node cần chạy container (HAProxy, Web01, Web02, Database). Control Node về nguyên tắc chỉ cần Ansible — NHƯNG nếu bạn làm theo khuyến nghị ở Bước 6 (đặt Prometheus + Grafana trên Control Node), node này cũng cần cài Docker Engine để chạy stack giám sát đó; nếu muốn giữ Control Node "sạch" hoàn toàn không Docker, hãy đổi Bước 6 sang triển khai monitoring trên một VM riêng thay vì Control Node.

## Kiến trúc yêu cầu

Tuân thủ đúng sơ đồ ở [[03-Architecture]]:

- **VM1 — Control Node**: cài Ansible, chứa toàn bộ inventory, playbook, roles, template của dự án. Có SSH key riêng cho dự án (không dùng chung key cá nhân), kết nối tới 4 node còn lại qua public key authentication, không dùng mật khẩu.
- **VM2 — HAProxy**: chạy HAProxy làm Load Balancer / reverse proxy ở tầng trước cùng, lắng nghe port 80 (và 443 nếu bạn muốn tự cấu hình TLS bằng kiến thức đã học ở Module 16), phân phối traffic sang Web01/Web02 kèm health check chủ động.
- **VM3, VM4 — Web01, Web02**: mỗi node chạy Docker Compose với tối thiểu một container Nginx phục vụ ứng dụng, đảm bảo tầng này hoàn toàn stateless (không lưu session cục bộ — mọi trạng thái dùng chung đẩy về Redis trên VM5).
- **VM5 — Database**: chạy Docker Compose với 3 container riêng biệt — MySQL (dữ liệu chính), Redis (cache/session dùng chung cho Web01/Web02), RabbitMQ (message queue cho tác vụ bất đồng bộ). Mỗi service có volume riêng, không được public trực tiếp ra ngoài mạng nội bộ dự án.

## Các bước triển khai

### Bước 1 — Chuẩn bị SSH và Inventory

Tạo SSH key riêng cho dự án trên Control Node, phân phối public key sang cả 4 managed node. Trên mỗi managed node, cấu hình user dùng cho Ansible có quyền `sudo` không cần nhập mật khẩu cho các lệnh cần thiết (thêm dòng NOPASSWD vào `/etc/sudoers.d/`, theo đúng kiến thức Least Privilege đã học ở khóa Linux Sysadmin) — bắt buộc phải có bước này, vì nếu không, `become: true` trong playbook sẽ đứng chờ nhập mật khẩu và mọi tác vụ chạy tự động không giám sát (ví dụ cron backup ở Bước 7) sẽ thất bại. Viết file `inventory.ini` phân nhóm rõ ràng theo vai trò (`haproxy`, `web`, `database`), đúng cấu trúc tham khảo ở [[04-Commands]]. Xác nhận kết nối thành công bằng `ansible all -m ping` trước khi viết bất kỳ playbook nào.

### Bước 2 — Thiết kế Roles

Viết tối thiểu 4 role riêng biệt: `common` (cấu hình nền tảng chung cho mọi node — timezone, user, firewall cơ bản), `docker` (cài đặt Docker Engine, áp dụng lại một phần kiến thức hardening từ Module 18 — ví dụ cấu hình log rotation mặc định từ Module 17 ngay trong role này), `haproxy` (cài đặt và cấu hình HAProxy bằng template Jinja2), `webapp` và `database` (deploy đúng Docker Compose stack tương ứng vai trò).

### Bước 3 — Viết Playbook chính

Playbook `site.yml` gọi đúng role cho đúng nhóm host, tận dụng cấu trúc `hosts: <group>` để mỗi node chỉ nhận role phù hợp với vai trò của nó — không role nào được áp dụng nhầm sang node không liên quan.

### Bước 4 — Template hóa cấu hình HAProxy

Không hard-code danh sách IP của Web01/Web02 trực tiếp trong file cấu hình — dùng template Jinja2 (`haproxy.cfg.j2`) đọc từ chính `inventory.ini` (biến `groups['web']`), để khi cần thêm Web03 trong tương lai, chỉ cần thêm một dòng vào inventory và chạy lại playbook, không cần sửa tay file cấu hình HAProxy.

### Bước 5 — Deploy Docker Compose cho từng vai trò

Web01/Web02 deploy Nginx phục vụ nội dung tĩnh hoặc reverse-proxy tới một ứng dụng backend đơn giản do bạn tự chọn (có thể tái sử dụng ứng dụng mẫu đã dùng ở các lab Docker Compose trước đó, Module 14-15). Database deploy đủ 3 service MySQL/Redis/RabbitMQ với volume riêng cho từng service, network Docker Compose nội bộ, không expose port ra ngoài trừ những port thật sự cần thiết cho vận hành (ví dụ RabbitMQ management UI chỉ nên truy cập qua mạng nội bộ, không public).

### Bước 6 — Gắn Monitoring

Chọn một VM (khuyến nghị Control Node, để không cạnh tranh tài nguyên với service đang phục vụ traffic thật) triển khai Prometheus + Grafana theo kiến thức Module 17, cấu hình scrape cAdvisor chạy trên cả 4 node còn lại (mỗi node cần tự chạy một instance cAdvisor riêng, xem lại Module 17 nếu quên cách deploy). Xây dựng tối thiểu một dashboard tổng hợp hiển thị được tình trạng tài nguyên của toàn bộ 4 node cùng lúc, không chỉ từng node riêng lẻ.

### Bước 7 — Thiết lập Backup

Thiết lập một cơ chế backup tự động (cron job gọi script, hoặc Ansible playbook riêng chạy định kỳ) thực hiện cả hai loại backup đã học ở [[02-Theory]]: rsync đồng bộ file cấu hình/mã nguồn tĩnh từ Web01/Web02 về một nơi lưu trữ tập trung, và volume backup đóng gói dữ liệu MySQL theo lịch định kỳ. Ghi rõ RPO/RTO bạn tự đặt ra cho dự án (ví dụ RPO 24h nếu backup mỗi ngày một lần) và giải thích vì sao mức đó phù hợp với một ứng dụng nội bộ quy mô nhỏ.

### Bước 8 — Kiểm thử toàn bộ hệ thống

Xác nhận toàn bộ luồng hoạt động đúng như kiến trúc: gọi vào IP/domain của HAProxy thấy được nội dung ứng dụng, tắt một trong hai Web server để xác nhận HAProxy tự động loại nó khỏi vòng xoay (health check hoạt động đúng), xác nhận dữ liệu ghi từ Web01 đọc lại được từ Web02 (thông qua Database dùng chung), xác nhận dashboard Grafana hiển thị đúng số liệu của cả 4 node, xác nhận file backup được tạo ra đúng lịch đã cấu hình.

## Tiêu chí hoàn thành

- Toàn bộ 5 VM được triển khai **hoàn toàn bằng Ansible** — không có bước cấu hình thủ công nào ngoài việc tạo VM ban đầu và cài hệ điều hành gốc. Chạy lại `ansible-playbook site.yml` từ đầu trên VM sạch phải cho ra đúng kết quả hạ tầng tương tự (tính idempotent — khái niệm đã học ở Module 02).
- HAProxy phân phối traffic thành công tới cả Web01 và Web02, tự động phát hiện và loại một node khi node đó ngừng phản hồi.
- Ứng dụng web đọc/ghi được dữ liệu nhất quán qua Database dùng chung (MySQL và/hoặc Redis), bất kể request rơi vào Web01 hay Web02.
- Có dashboard Grafana hiển thị được tình trạng tài nguyên của toàn bộ 4 node hạ tầng (không chỉ 1 node).
- Có bằng chứng cụ thể (file backup thực tế, log cron job chạy thành công) cho cả rsync và volume backup, kèm giải thích RPO/RTO tự đặt ra.
- Toàn bộ mã nguồn dự án (Ansible role, playbook, template, Docker Compose file, script backup) được lưu lại có tổ chức trong [[labs/README|labs/]] của module này, đủ để một người khác có thể clone và triển khai lại từ đầu chỉ bằng cách đọc README.
