---
title: "02 - Theory"
module: 7
tags: [ansible, sysops-infra, module-07, multi-tier, rolling-update, zero-downtime, haproxy, backup, monitoring]
---

# 02 — Lý thuyết

## 1. Multi-tier Architecture — vì sao tách tầng

Kiến trúc doanh nghiệp điển hình tách thành nhiều tầng, mỗi tầng có nhóm inventory riêng (đã luyện tư duy từ Module 01-02):

| Tầng | Vai trò | Đặc thù triển khai |
|---|---|---|
| Load Balancer | Phân phối traffic, health check | Thường ít node (2 cho HA), thay đổi cấu hình nhạy cảm — sai sót ảnh hưởng toàn bộ hệ thống |
| Web/App tier | Xử lý logic ứng dụng | Nhiều node, chịu tải chính — mục tiêu chính của Rolling Update |
| Database tier | Lưu trữ dữ liệu | Rất ít node, không thể "rolling update" tùy tiện — cần chiến lược riêng (primary/replica, backup trước khi thay đổi schema) |

Tách tầng bằng inventory cho phép áp dụng **chiến lược triển khai khác nhau** cho từng tầng trong cùng một Playbook lớn — ví dụ Web tier dùng `serial: "20%"`, nhưng Database tier chỉ chạy tuần tự từng node một với thời gian chờ (`pause`) dài hơn nhiều để đảm bảo đồng bộ dữ liệu.

## 2. Rolling Update — `serial` và `max_fail_percentage`

`serial` giới hạn số host xử lý trong mỗi "đợt" (batch) của một Play, thay vì Ansible mặc định chạy song song tất cả host cùng lúc (giới hạn bởi `forks`):

```yaml
- name: Rolling update web tier
  hosts: webservers
  serial: 2          # Chi cap nhat 2 host moi dot
  max_fail_percentage: 20   # Dung playbook neu > 20% tong so host that bai
  tasks:
    - name: Cap nhat ma nguon ung dung
      ansible.builtin.git:
        repo: "https://git.example.com/app.git"
        dest: /opt/myapp
      notify: Restart app
```

**`serial` hỗ trợ 3 dạng khai báo:**
- **Số cố định:** `serial: 2` — luôn xử lý 2 host mỗi đợt.
- **Phần trăm:** `serial: "25%"` — xử lý 25% tổng số host mỗi đợt (tính trên tổng host của Play, không phải tổng host còn lại).
- **Progressive (canary deployment):** `serial: [1, 5, "25%", "100%"]` — đợt đầu chỉ 1 host (kiểm chứng thay đổi an toàn trên phạm vi nhỏ nhất), đợt hai 5 host, đợt ba 25%, đợt cuối phần còn lại. Đây là mô hình "canary" phổ biến trong triển khai doanh nghiệp thực tế: rủi ro thấp nhất được đặt lên trước.

**`max_fail_percentage`** tính theo **tổng số host trong toàn bộ Play**, không phải theo từng đợt `serial` — nếu tỷ lệ thất bại tích lũy vượt ngưỡng, Ansible dừng toàn bộ Playbook ngay, các host chưa được xử lý giữ nguyên phiên bản cũ, giới hạn phạm vi ảnh hưởng (blast radius) của một lỗi triển khai.

## 3. Zero Downtime Deployment — phối hợp với Load Balancer

Chỉ dùng `serial` là **chưa đủ** cho zero downtime thật sự — cần phối hợp với Load Balancer để đảm bảo traffic không bao giờ được gửi tới một node đang trong quá trình cập nhật. Quy trình chuẩn:

```yaml
- name: Zero downtime rolling update
  hosts: webservers
  serial: 1
  tasks:
    - name: Buoc 1 - Go node khoi pool Load Balancer (HAProxy)
      ansible.builtin.command: >
        echo "disable server webapp/{{ inventory_hostname }}" | socat stdio /var/run/haproxy.sock
      delegate_to: "{{ groups['loadbalancers'][0] }}"

    - name: Buoc 2 - Cho ket noi dang xu ly hoan tat (drain)
      ansible.builtin.pause:
        seconds: 15

    - name: Buoc 3 - Cap nhat ung dung tren node nay
      ansible.builtin.git:
        repo: "https://git.example.com/app.git"
        dest: /opt/myapp

    - name: Buoc 4 - Restart app
      ansible.builtin.systemd:
        name: myapp
        state: restarted

    - name: Buoc 5 - Health check truoc khi dua lai vao pool
      ansible.builtin.uri:
        url: "http://{{ inventory_hostname }}/healthz"
        status_code: 200
      register: health_result
      until: health_result.status == 200
      retries: 5
      delay: 3

    - name: Buoc 6 - Dua node tro lai pool Load Balancer
      ansible.builtin.command: >
        echo "enable server webapp/{{ inventory_hostname }}" | socat stdio /var/run/haproxy.sock
      delegate_to: "{{ groups['loadbalancers'][0] }}"
      when: health_result.status == 200
```

**Giải thích các kỹ thuật mới xuất hiện:**
- **`delegate_to`:** chạy Task này **trên một host khác** thay vì host hiện tại của vòng lặp (`inventory_hostname`) — ở đây, lệnh điều khiển HAProxy phải chạy trên chính máy Load Balancer, không phải trên web server đang được cập nhật.
- **`until`/`retries`/`delay`:** cơ chế retry có sẵn của Ansible — thử lại Task tối đa `retries` lần, cách nhau `delay` giây, cho tới khi điều kiện `until` đúng. Đây là cách triển khai health check "chờ ứng dụng thật sự sẵn sàng" thay vì tin tưởng mù quáng service đã `started` là đã sẵn sàng nhận traffic.
- **`when: health_result.status == 200`:** chỉ đưa node trở lại pool nếu health check thực sự thành công — nếu không, node bị bỏ lại ngoài pool (an toàn hơn là đưa một node lỗi vào phục vụ traffic thật).

## 4. HAProxy/Nginx làm Load Balancer — cấu hình qua Template

Load Balancer bản thân cũng là một dịch vụ được Ansible quản lý, cấu hình qua `template` (Module 04):

```jinja2
{# haproxy.cfg.j2 #}
backend webapp
    balance roundrobin
    option httpchk GET /healthz
{% for host in groups['webservers'] %}
    server {{ host }} {{ hostvars[host]['ansible_default_ipv4']['address'] }}:80 check
{% endfor %}
```

Điểm đáng chú ý: `{% for host in groups['webservers'] %}` lặp qua **toàn bộ host trong một nhóm inventory** (biến `groups` là biến "magic" Ansible tự cung cấp), kết hợp `hostvars[host]` để lấy Facts/Variables của **host khác**, không phải host hiện tại đang chạy Play — kỹ thuật quan trọng để một node (Load Balancer) "biết" về toàn bộ node khác (Web tier) mà không cần khai báo IP thủ công.

## 5. Database Tier — vì sao không "rolling update" như Web tier

Database tier có ràng buộc khác biệt căn bản: dữ liệu **không thể** mất hoặc không đồng nhất, và thường chỉ có 1 node ghi (primary) tại một thời điểm. Nguyên tắc triển khai:

- **Không bao giờ áp dụng `serial` giống Web tier** cho thay đổi schema/cấu hình primary — mọi thay đổi cấu trúc dữ liệu cần kế hoạch riêng (thường ngoài giờ cao điểm, có backup trước, có kế hoạch rollback rõ ràng).
- **Backup bắt buộc trước mọi thay đổi** liên quan schema hoặc nâng cấp phiên bản — không có "chạy lại Playbook" đơn giản như restart một web server.
- **Failover** (chuyển primary sang replica khi primary lỗi) là quy trình vận hành riêng biệt, thường có công cụ chuyên dụng hỗ trợ (Patroni, repmgr cho PostgreSQL) — Ansible đóng vai trò điều phối các bước, không tự phát minh logic failover.

## 6. Backup Strategy tích hợp vào Playbook

```yaml
- name: Backup database truoc khi trien khai
  hosts: dbservers
  tasks:
    - name: Tao ban backup co timestamp
      ansible.builtin.command: >
        pg_dump -U postgres mydb -f /backup/mydb-{{ ansible_date_time.iso8601_basic_short }}.sql
      register: backup_result

    - name: Xac nhan file backup duoc tao thanh cong va khong rong
      ansible.builtin.stat:
        path: "/backup/mydb-{{ ansible_date_time.iso8601_basic_short }}.sql"
      register: backup_file
      failed_when: not backup_file.stat.exists or backup_file.stat.size == 0
```

Biến `ansible_date_time.iso8601_basic_short` là một Fact (Module 02) — dùng Facts để tạo tên file có timestamp là cách chuẩn, tránh lỗi idempotency đã cảnh báo ở Module 04 (không dùng `lookup('pipe', 'date')` render trực tiếp vào nội dung cấu hình, nhưng dùng cho **tên file** là phù hợp vì đây chính là mục đích cần giá trị thay đổi mỗi lần).

## 7. Monitoring Deployment — giám sát chính quy trình triển khai

Ngoài giám sát hạ tầng thường trực (ngoài phạm vi khóa này), Playbook triển khai nên tự báo cáo trạng thái của chính nó:

```yaml
- name: Thong bao trien khai thanh cong
  ansible.builtin.uri:
    url: "https://monitoring.example.com/api/events"
    method: POST
    body_format: json
    body:
      event: "deployment_success"
      host: "{{ inventory_hostname }}"
      version: "{{ app_version }}"
  delegate_to: localhost
  run_once: true
```

`run_once: true` đảm bảo Task này chỉ chạy **một lần duy nhất** cho toàn bộ Play (dù Play nhắm vào nhiều host) — phù hợp cho hành động mang tính tổng kết (gửi một thông báo duy nhất về việc triển khai đã hoàn tất), tránh gửi trùng lặp N lần cho N host.
