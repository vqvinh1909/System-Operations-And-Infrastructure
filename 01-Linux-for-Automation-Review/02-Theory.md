---
title: "02 - Theory"
module: 1
tags: [ansible, sysops-infra, module-01, ssh, sudo, yaml, jinja2, inventory]
---

# 02 — Lý thuyết

## 1. SSH dưới góc nhìn automation

Ở khóa Linux Sysadmin, bạn học SSH để **bạn** (con người) đăng nhập từ xa. Với Ansible, SSH là **kênh giao tiếp duy nhất** giữa control node và managed node — mọi module Python được đẩy qua kênh này. Điều đó kéo theo 3 yêu cầu bắt buộc, khác với thói quen dùng SSH cá nhân:

- **Bắt buộc dùng key-based authentication, không dùng password.** Ansible có thể chạy hàng trăm task liên tiếp trên hàng chục host — không ai ngồi gõ password cho từng kết nối. Nếu SSH server còn cho phép `PasswordAuthentication yes`, đây vừa là rủi ro bảo mật (đã học ở Module SSH Hardening) vừa là rào cản automation.
- **Cần một cặp key riêng cho automation**, tách biệt với key cá nhân bạn dùng để tự SSH vào server làm việc hàng ngày. Lý do: key automation thường được lưu trên CI/CD runner hoặc control node dùng chung cho cả team — nếu dùng chung key cá nhân, bạn không thể thu hồi (revoke) quyền của một người mà không ảnh hưởng tới automation đang chạy.
- **`known_hosts` phải được xử lý nhất quán.** Lần đầu SSH tới một host mới, hệ thống hỏi xác nhận fingerprint — hành vi tương tác này sẽ làm Ansible treo (hang) nếu chạy tự động (ví dụ trong CI/CD). Cần cấu hình trước (`ssh-keyscan` thêm vào `known_hosts`, hoặc set biến `ANSIBLE_HOST_KEY_CHECKING=False` cho môi trường lab/test — **không khuyến nghị tắt hoàn toàn ở production** vì mất khả năng phát hiện tấn công man-in-the-middle).

## 2. SSH Config — quản lý nhiều host có tổ chức

File `~/.ssh/config` cho phép đặt alias và cấu hình riêng cho từng host/nhóm host, thay vì phải nhớ và gõ đầy đủ `ssh -i ~/.ssh/id_automation -p 2222 deploy@10.0.1.15` mỗi lần. Đây là bước đệm tư duy trực tiếp cho khái niệm **Inventory** ở Module 02 — cả hai đều là cách "đặt tên gọn cho một nhóm máy có đặc điểm chung".

```
# ~/.ssh/config
Host web-*
    User deploy
    IdentityFile ~/.ssh/id_automation
    StrictHostKeyChecking accept-new

Host web-01
    HostName 10.0.1.11

Host web-02
    HostName 10.0.1.12

Host db-01
    HostName 10.0.1.20
    User dbadmin
    Port 2222
    IdentityFile ~/.ssh/id_automation
```

Với cấu hình này, `ssh web-01` tự động dùng đúng user, key, và các tham số kế thừa từ khối `Host web-*`. Ansible **không bắt buộc** phải dùng `~/.ssh/config` (nó có cách khai báo `ansible_user`, `ansible_ssh_private_key_file` riêng trong inventory — học ở Module 02), nhưng hiểu cơ chế kế thừa cấu hình theo pattern này giúp bạn hiểu ngay logic `group_vars`/`host_vars` sau này — chúng giải quyết cùng một bài toán theo tinh thần tương tự.

## 3. Sudo cho Service Account (automation)

Ở khóa Linux Sysadmin, bạn học cấu hình sudo cho **user thật** — một kỹ sư cần quyền hạn chế theo vai trò. Với automation, có thêm một khái niệm: **service account** — một user hệ thống không đại diện cho một người cụ thể, mà đại diện cho **Ansible chạy thay mặt team**.

Nguyên tắc thiết kế sudo cho service account:

- **NOPASSWD có kiểm soát:** Ansible cần chạy nhiều task với `become: true` (sudo) liên tiếp không dừng lại chờ nhập password. Cấu hình `NOPASSWD` trong `/etc/sudoers.d/` cho user automation — nhưng **giới hạn phạm vi lệnh** nếu có thể, thay vì cho `ALL=(ALL) NOPASSWD: ALL` vô điều kiện.
- **Tên user rõ ràng, dễ audit:** đặt tên như `ansible_svc` hoặc `deploy`, không dùng `root` trực tiếp — để log `sudo` (`/var/log/secure` hoặc `/var/log/auth.log`, đã học ở khóa Linux Sysadmin) ghi rõ hành động nào tới từ automation.
- **Key riêng, quyền riêng:** service account này chỉ nên đăng nhập bằng SSH key automation (mục 1), không có password đăng nhập trực tiếp thông thường.

Ví dụ file `/etc/sudoers.d/ansible`:

```
# Cho phep service account 'deploy' chay moi lenh qua sudo ma khong hoi password,
# phuc vu Ansible voi become: true
deploy ALL=(ALL) NOPASSWD: ALL
```

> Đây là cấu hình đơn giản dùng cho lab/học tập. Ở môi trường doanh nghiệp trưởng thành, đội bảo mật thường giới hạn `NOPASSWD` chỉ cho một số lệnh cụ thể cần thiết, áp dụng logging chi tiết hơn (`Defaults log_output` với `sudosh`/`auditd`) — nội dung này thuộc phạm vi khóa Linux Sysadmin Module Security Hardening, module này chỉ ôn lại phần liên quan trực tiếp tới Ansible.

## 4. Inventory Planning — tư duy trước khi viết code

Trước khi mở Module 02 và viết file inventory Ansible đầu tiên, bạn cần **lập kế hoạch trên giấy** cách nhóm host — giống việc thiết kế trước khi code. Ba trục phân loại phổ biến trong doanh nghiệp:

- **Theo môi trường (environment):** `production`, `staging`, `development` — tách biệt hoàn toàn, tránh nguy cơ chạy nhầm playbook vào production.
- **Theo vai trò (role):** `webservers`, `dbservers`, `loadbalancers` — nhóm theo chức năng để áp dụng đúng cấu hình cho đúng loại máy.
- **Theo vị trí địa lý/datacenter (nếu có):** `dc-hanoi`, `dc-hcm`, hoặc theo region cloud (`ap-southeast-1`) — quan trọng khi hạ tầng trải nhiều vùng.

Một host thường thuộc **nhiều nhóm cùng lúc** (ví dụ `web-01` vừa thuộc `production`, vừa thuộc `webservers`, vừa thuộc `dc-hanoi`) — đây chính là mô hình inventory thật của Ansible sẽ học ở Module 02. Việc luyện tư duy phân nhóm này trước giúp bạn không bị choáng ngợp khi thấy cấu trúc `group_vars/`, `host_vars/` lần đầu.

## 5. YAML Basics

**YAML (YAML Ain't Markup Language)** là định dạng dữ liệu mà **toàn bộ Ansible** (playbook, inventory dạng YAML, role, vault file) được viết bằng nó. Không hiểu đúng YAML, bạn sẽ liên tục gặp lỗi cú pháp khó hiểu ở các module sau.

### Nguyên tắc cốt lõi

- **Indentation (thụt lề) có ý nghĩa cấu trúc** — giống Python, không dùng tab, chỉ dùng space (khuyến nghị 2 space mỗi cấp, đây là chuẩn phổ biến trong cộng đồng Ansible).
- **Scalar (giá trị đơn):** chuỗi, số, boolean.

```yaml
name: "web-01"
port: 8080
is_production: true
```

- **List (danh sách):** mỗi phần tử bắt đầu bằng dấu `-`, cùng cấp thụt lề.

```yaml
packages:
  - nginx
  - git
  - curl
```

- **Dictionary (mapping / key-value):** thụt lề để thể hiện quan hệ cha-con.

```yaml
server:
  name: web-01
  ip: 10.0.1.11
  port: 8080
```

- **Kết hợp list + dictionary** (mẫu hình bạn sẽ thấy suốt các module Playbook/Role):

```yaml
users:
  - name: alice
    groups: [sudo, docker]
  - name: bob
    groups: [docker]
```

- **Comment:** bắt đầu bằng `#`.
- **Document separator:** `---` ở đầu file (Ansible playbook luôn bắt đầu bằng `---`).
- **Lỗi thường gặp nhất:** trộn lẫn tab và space, hoặc thụt lề sai 1 space cũng khiến YAML hiểu sai cấu trúc (khác cấp cha-con) mà không báo lỗi rõ ràng — sẽ đi sâu ở [[06-Troubleshooting|06 - Troubleshooting]].

## 6. Jinja2 Basics

**Jinja2** là template engine mà Ansible dùng để tạo giá trị động (dynamic) trong playbook và file cấu hình. Học sâu ở Module 04 — ở đây chỉ cần nhận diện 3 loại cú pháp cơ bản:

| Cú pháp | Ý nghĩa | Ví dụ |
|---|---|---|
| `{{ ... }}` | In ra giá trị của biến/biểu thức | `{{ ansible_hostname }}` in ra tên host |
| `{% ... %}` | Câu lệnh điều khiển (loop, if) — dùng trong template file, không dùng trực tiếp trong YAML task | `{% if env == "prod" %} ... {% endif %}` |
| `{# ... #}` | Comment (không xuất hiện trong output) | `{# ghi chu, khong hien ra #}` |

Ví dụ một dòng trong playbook dùng Jinja2 để tham chiếu biến:

```yaml
- name: In ra ten server hien tai
  ansible.builtin.debug:
    msg: "Server nay ten la {{ inventory_hostname }}, chay o moi truong {{ env }}"
```

Ghi nhớ: mọi giá trị YAML bắt đầu bằng `{{ ` phải được **đặt trong dấu ngoặc kép** cả biểu thức (`msg: "{{ bien }}"`), nếu không YAML parser sẽ hiểu nhầm `{{ ` là ký hiệu mở một dictionary YAML (`{ }`) chứ không phải Jinja2 — lỗi cú pháp rất phổ biến với người mới, sẽ minh họa ở [[06-Troubleshooting|06 - Troubleshooting]].
