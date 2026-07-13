---
title: "02 - Theory"
module: 2
tags: [ansible, sysops-infra, module-02, architecture, inventory, variables, facts, modules]
---

# 02 — Lý thuyết

## 1. Kiến trúc Agentless — vì sao quan trọng

Ansible **không cần cài phần mềm agent thường trực** trên managed node. Mọi tác vụ diễn ra theo chu trình: control node kết nối SSH tới managed node → đẩy (push) một module Python nhỏ, tự chứa (self-contained) → module thực thi tại chỗ trên managed node → trả kết quả dưới dạng JSON về control node → module bị xóa khỏi managed node.

Hệ quả thực tế trong doanh nghiệp:
- **Không có agent nghĩa là không có thêm một tiến trình chạy nền tiêu tốn RAM/CPU** trên hàng trăm server, không có thêm một daemon cần patch bảo mật định kỳ, không có thêm một cổng mạng (port) phải mở cho agent giao tiếp về master.
- **Điều kiện cần trên managed node chỉ là:** SSH server đang chạy (gần như mọi Linux server đều có sẵn) và Python đã cài (cũng thường có sẵn hoặc dễ cài). Đây là lý do một server mới tinh có thể được Ansible quản lý ngay mà không cần bước "đăng ký" phức tạp như mô hình agent-based (Puppet/Chef).
- **Đánh đổi:** với hàng nghìn server, overhead thiết lập kết nối SSH cho mỗi task có thể chậm hơn agent thường trực đã kết nối sẵn — đây là lý do Ansible có cơ chế tối ưu như `pipelining`, `mitogen` (bên thứ ba), và `ansible-pull` (đảo chiều mô hình, sẽ nhắc ở Module 07).

## 2. Control Node

Control Node là máy **duy nhất** cần cài Ansible — nơi bạn gõ lệnh `ansible`, `ansible-playbook`. Yêu cầu:

- Hệ điều hành: Linux/Unix/macOS. **Windows không thể làm control node trực tiếp** (chỉ có thể làm managed node qua WinRM) — nếu dùng Windows, cần WSL.
- Python: ansible-core nhánh mới nhất (2.21, tính tới thời điểm biên soạn giáo trình 07/2026) yêu cầu Python 3.12–3.14 trên control node.
- Kết nối mạng SSH tới toàn bộ managed node.

Có **thể có nhiều control node** trong tổ chức lớn (mỗi kỹ sư một control node cá nhân, cộng thêm CI/CD runner làm control node cho pipeline tự động) — miễn tất cả cùng đọc chung một nguồn inventory/playbook (thường qua Git, đúng tinh thần GitOps đã học ở Module 00).

## 3. Managed Node

Managed Node là máy **bị quản lý**, không cần cài Ansible. Yêu cầu tối thiểu:

- SSH server hoạt động, chấp nhận key-based authentication (đã chuẩn bị ở Module 01).
- Python đã cài (ansible-core 2.21 hỗ trợ managed node Python 3.9–3.14). Nếu managed node không có Python (hiếm, ví dụ image tối giản), Ansible có `raw` module hoạt động không cần Python để cài Python trước.
- User có quyền sudo phù hợp nếu task cần quyền root (`become: true`).

## 4. Inventory — danh sách host và cách tổ chức

Inventory là nguồn dữ liệu Ansible dùng để biết **host nào tồn tại** và **host nào thuộc nhóm nào**. Có 4 dạng:

### 4.1 Static Inventory (INI)

```ini
[webservers]
web-01 ansible_host=10.0.1.11
web-02 ansible_host=10.0.1.12

[dbservers]
db-01 ansible_host=10.0.1.20 ansible_user=dbadmin

[production:children]
webservers
dbservers
```

`[production:children]` tạo một nhóm cha gộp nhiều nhóm con — cùng ý tưởng "một host thuộc nhiều nhóm" đã luyện tư duy ở Module 01.

### 4.2 Static Inventory (YAML) — khuyến nghị cho dự án mới

```yaml
all:
  children:
    webservers:
      hosts:
        web-01:
          ansible_host: 10.0.1.11
        web-02:
          ansible_host: 10.0.1.12
    dbservers:
      hosts:
        db-01:
          ansible_host: 10.0.1.20
          ansible_user: dbadmin
    production:
      children:
        webservers:
        dbservers:
```

YAML dễ đọc hơn khi inventory phức tạp, dễ tích hợp với `group_vars`/`host_vars` (biến gắn theo nhóm/host, sẽ đi sâu ở Module 05), và nhất quán về định dạng với Playbook.

### 4.3 Dynamic Inventory (script)

Với hạ tầng cloud thay đổi liên tục (auto-scaling, container ngắn hạn), duy trì inventory tĩnh thủ công là bất khả thi. **Dynamic Inventory** là một script (executable, có quyền thực thi) trả về JSON theo schema Ansible quy định khi được gọi bằng `--list`. Ansible gọi script này mỗi lần chạy để lấy danh sách host **tại thời điểm hiện tại**, thay vì đọc file tĩnh.

### 4.4 Inventory Plugin — Cloud (AWS/Azure/GCP), giới thiệu

Từ Ansible 2.x trở đi, cách hiện đại thay cho tự viết dynamic inventory script là dùng **Inventory Plugin** có sẵn trong các collection cloud chính thức:

| Cloud | Collection | Plugin inventory |
|---|---|---|
| AWS | `amazon.aws` | `aws_ec2` |
| Azure | `azure.azcollection` | `azure_rm` |
| GCP | `google.cloud` | `gcp_compute` |

Ví dụ khai báo file `aws_ec2.aws_ec2.yml` (Ansible tự nhận diện qua đuôi tên file):

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - ap-southeast-1
filters:
  instance-state-name: running
keyed_groups:
  - key: tags.Environment
    prefix: env
```

Plugin tự động truy vấn API cloud provider, nhóm host theo tag (ví dụ instance có tag `Environment: production` tự vào nhóm `env_production`). Phạm vi khóa này chỉ giới thiệu khái niệm và cú pháp nền tảng — triển khai plugin cloud cụ thể theo từng provider (IAM permission, credential chain) thuộc phạm vi chuyên sâu hơn ngoài khóa.

## 5. Variables — biến do người dùng định nghĩa

Variables là giá trị bạn **chủ động khai báo** để tham số hóa playbook/inventory — ví dụ tên gói cần cài, cổng dịch vụ, phiên bản phần mềm. Có thể khai báo ở nhiều nơi (inventory, `group_vars/`, `host_vars/`, trong playbook, dòng lệnh `--extra-vars`) — thứ tự ưu tiên chi tiết (variable precedence) sẽ học sâu ở Module 05, module này chỉ cần biết khái niệm và một ví dụ khai báo trong inventory YAML:

```yaml
webservers:
  hosts:
    web-01:
      ansible_host: 10.0.1.11
      http_port: 8080
      app_version: "2.4.1"
```

## 6. Facts — dữ liệu Ansible tự thu thập

Khác với Variables (bạn viết ra), **Facts** là dữ liệu Ansible **tự động thu thập** từ managed node khi kết nối, thông qua module đặc biệt tên `setup`. Facts bao gồm: hostname, địa chỉ IP, hệ điều hành, phiên bản kernel, dung lượng RAM, số CPU, danh sách interface mạng, và hàng trăm thuộc tính khác.

Xem toàn bộ facts của một host:
```bash
ansible web-01 -m ansible.builtin.setup
```

Facts được dùng lại trong Playbook qua biến có tiền tố `ansible_`, ví dụ `ansible_hostname`, `ansible_distribution`, `ansible_default_ipv4.address`. Đây là cơ chế cho phép viết một playbook **chạy đúng trên nhiều hệ điều hành khác nhau** (ví dụ chọn tên package `httpd` trên RHEL hay `apache2` trên Debian dựa vào `ansible_os_family`) — nền tảng cho tính linh hoạt sẽ khai thác sâu ở Module 03 (Conditionals).

> Facts gathering tốn thời gian (mỗi lần chạy phải SSH thu thập lại toàn bộ). Với playbook không cần facts, có thể tắt bằng `gather_facts: false` để tăng tốc — chi tiết ở Module 03.

## 7. Modules — đơn vị thực thi nhỏ nhất

**Module** là một đoạn code (thường Python) thực hiện một tác vụ cụ thể, idempotent, nhận tham số đầu vào, trả kết quả JSON có cấu trúc (bao gồm trạng thái `changed: true/false`). Ansible đi kèm hàng nghìn module built-in (namespace `ansible.builtin`) và hàng chục nghìn module khác qua Collections (namespace theo nhà cung cấp, ví dụ `community.general`, `amazon.aws`).

Ví dụ nhóm module hay dùng nhất:

| Nhóm | Module ví dụ | Chức năng |
|---|---|---|
| Kết nối/kiểm tra | `ansible.builtin.ping` | Kiểm tra kết nối tới managed node (không phải ICMP ping) |
| Quản lý package | `ansible.builtin.apt`, `ansible.builtin.yum`, `ansible.builtin.dnf` | Cài/gỡ gói phần mềm |
| Quản lý file | `ansible.builtin.copy`, `ansible.builtin.file`, `ansible.builtin.template` | Sao chép, đặt quyền, render template |
| Quản lý service | `ansible.builtin.service`, `ansible.builtin.systemd` | Start/stop/enable dịch vụ |
| Thực thi lệnh | `ansible.builtin.command`, `ansible.builtin.shell` | Chạy lệnh trực tiếp (chỉ dùng khi không có module chuyên biệt) |
| Thu thập thông tin | `ansible.builtin.setup` | Thu thập facts |

**Nguyên tắc quan trọng:** luôn ưu tiên module chuyên biệt (`apt`, `copy`, `service`...) thay vì `command`/`shell`, vì module chuyên biệt tự đảm bảo idempotency (kiểm tra trạng thái trước khi hành động), còn `command`/`shell` chạy lệnh vô điều kiện mỗi lần — vi phạm nguyên lý idempotent đã học ở Module 00.

## 8. Ad-hoc Command — chạy module trực tiếp không cần Playbook

Ad-hoc command là cách gọi một module **một lần**, trực tiếp từ dòng lệnh, không cần viết file YAML. Cú pháp chung:

```bash
ansible <pattern-host-hoac-nhom> -m <module> -a "<tham-so>"
```

Ví dụ: `ansible webservers -m ansible.builtin.ping`, hoặc `ansible all -m ansible.builtin.command -a "uptime"`.

**Khi nào dùng ad-hoc, khi nào cần Playbook (Module 03):**

| Tình huống | Nên dùng |
|---|---|
| Kiểm tra nhanh, một lần, không cần lặp lại | Ad-hoc |
| Tác vụ khẩn cấp (restart service trên toàn bộ fleet) | Ad-hoc |
| Cần nhiều bước tuần tự, có điều kiện, có xử lý lỗi | Playbook |
| Cần lưu lại, review, chạy lại nhiều lần, đưa vào CI/CD | Playbook |

Quy tắc thực tế: nếu bạn thấy mình gõ lại **cùng một ad-hoc command lần thứ hai**, đó là dấu hiệu nên chuyển sang Playbook.

## 9. Configuration — `ansible.cfg`

`ansible.cfg` là file cấu hình trung tâm điều khiển hành vi mặc định của Ansible (inventory path, SSH user mặc định, số kết nối song song `forks`, timeout, v.v.). Ansible tìm file này theo **thứ tự ưu tiên cố định**, dừng ở file đầu tiên tìm thấy:

```
1. Bien moi truong ANSIBLE_CONFIG (tro thang toi mot file cu the)
2. ./ansible.cfg (thu muc hien tai noi chay lenh ansible)
3. ~/.ansible.cfg (thu muc home cua user)
4. /etc/ansible/ansible.cfg (cau hinh toan he thong)
```

Ví dụ `ansible.cfg` tối thiểu cho một dự án:

```ini
[defaults]
inventory = ./inventory.yml
remote_user = deploy
host_key_checking = True
forks = 10

[privilege_escalation]
become = True
become_method = sudo
```

Thực hành doanh nghiệp: đặt `ansible.cfg` **trong thư mục dự án** (ưu tiên #2), commit vào Git cùng playbook — đảm bảo mọi kỹ sư trong team và CI/CD runner đều dùng chung một cấu hình, tránh tình trạng "chạy trên máy tôi thì được".
