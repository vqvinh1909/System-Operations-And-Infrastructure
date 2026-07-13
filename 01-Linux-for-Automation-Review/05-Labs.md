---
title: "05 - Labs"
module: 1
tags: [ansible, sysops-infra, module-01, labs]
---

# 05 — Labs thực hành

> Yêu cầu môi trường: 1 máy control node (VM hoặc WSL) + tối thiểu 2 máy managed node (VM, container, hoặc cloud free-tier). Có thể dùng lại VM từ khóa Linux Sysadmin.

## Lab 1 — Tạo và triển khai SSH key riêng cho automation

**Mục tiêu:** có một kênh SSH key-based hoạt động, tách biệt với key cá nhân, sẵn sàng cho Ansible ở Module 02.

**Các bước:**

1. Trên control node, tạo cặp key riêng: `ssh-keygen -t ed25519 -f ~/.ssh/id_automation -C "ansible-automation-key"`.
2. Triển khai public key tới **ít nhất 2 managed node** bằng `ssh-copy-id`.
3. Xác nhận đăng nhập bằng key mới, không hỏi password, tới cả 2 node.
4. Chạy `ssh-keyscan` cho cả 2 node, thêm vào `known_hosts`, xác nhận không còn bị hỏi xác nhận fingerprint khi kết nối lần đầu qua script.

**Kết quả cần có:** file `~/.ssh/id_automation` + `id_automation.pub`, log kết nối thành công không cần password tới cả 2 managed node.

## Lab 2 — Cấu hình `~/.ssh/config` có tổ chức

**Mục tiêu:** quản lý nhiều host bằng alias thay vì nhớ IP/user/key thủ công.

**Các bước:**

1. Viết `~/.ssh/config` với 1 khối `Host web-*` chung (user, key, StrictHostKeyChecking) và 2 khối riêng cho từng host (chỉ định `HostName`).
2. Thêm 1 host thứ 3 giả lập là database server, dùng user và port khác (ví dụ `dbadmin`, port `2222`).
3. Dùng `ssh -G <alias>` để xác nhận cấu hình gộp đúng cho từng alias trước khi thử kết nối thật.
4. Kết nối thử cả 3 alias bằng lệnh ngắn gọn (`ssh web-01`, không cần thêm tham số).

**Kết quả cần có:** file `~/.ssh/config` hoàn chỉnh, output `ssh -G` cho cả 3 alias, log kết nối thành công.

## Lab 3 — Thiết lập service account + sudo cho automation

**Mục tiêu:** tạo user chuyên dụng cho automation với quyền sudo có kiểm soát.

**Các bước:**

1. Trên mỗi managed node, tạo user `deploy` (nếu chưa có).
2. Cài public key automation vào `~/.ssh/authorized_keys` của user `deploy`.
3. Tạo file `/etc/sudoers.d/ansible` cấp `NOPASSWD: ALL` cho `deploy`, kiểm tra cú pháp bằng `visudo -cf`.
4. Từ control node, test `ssh -i ~/.ssh/id_automation deploy@<ip> "sudo -n whoami"` — kỳ vọng trả về `root` không hỏi password.

**Kết quả cần có:** output `sudo -n whoami` trả về `root` trên cả 2 managed node.

## Lab 4 — Viết YAML inventory nháp (thủ công, chưa dùng Ansible)

**Mục tiêu:** luyện tư duy phân nhóm host trước khi học cú pháp inventory thật ở Module 02.

**Các bước:**

1. Viết một file `inventory-draft.yml` (thuần YAML, chưa cần đúng schema Ansible) mô tả các host lab của bạn, nhóm theo 3 trục: environment, role, datacenter — tối thiểu 4 host, mỗi host thuộc ít nhất 2 nhóm.
2. Chạy `python3 -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))" inventory-draft.yml` để xác nhận cú pháp hợp lệ.
3. Cố tình gõ sai thụt lề 1 chỗ, chạy lại lệnh kiểm tra, quan sát thông báo lỗi (giữ log lại để so sánh ở Lab 5).

**Kết quả cần có:** file `inventory-draft.yml` hợp lệ + 1 log lỗi cú pháp đã tạo ra có chủ đích.

## Lab 5 — Thử nghiệm Jinja2 cơ bản bằng Python

**Mục tiêu:** thấy Jinja2 hoạt động thật (không chỉ đọc lý thuyết) trước khi gặp lại trong playbook ở Module 03-04.

**Các bước:**

```bash
pip install jinja2
python3 << 'EOF'
from jinja2 import Template

t = Template("Server {{ name }} chay o moi truong {{ env }}")
print(t.render(name="web-01", env="production"))

t2 = Template("{% for pkg in packages %}{{ pkg }} {% endfor %}")
print(t2.render(packages=["nginx", "git", "curl"]))
EOF
```

1. Chạy đoạn script trên, quan sát output.
2. Sửa biến `env` thành giá trị khác, chạy lại.
3. Thêm 1 filter vào template (ví dụ `{{ name | upper }}`), chạy lại, quan sát kết quả.

**Kết quả cần có:** 3 lần output khác nhau đã lưu lại (copy vào `labs/` để đối chiếu sau này).

## Checklist hoàn thành module

- [ ] SSH key automation hoạt động, không hỏi password trên ≥ 2 managed node.
- [ ] `~/.ssh/config` có alias cho ≥ 3 host, xác nhận bằng `ssh -G`.
- [ ] Service account `deploy` có sudo NOPASSWD hoạt động, xác nhận bằng `sudo -n whoami`.
- [ ] Viết được YAML hợp lệ (list, dictionary, kết hợp) không cần tra cứu liên tục.
- [ ] Chạy thử Jinja2 render bằng Python, hiểu `{{ }}` khác `{% %}` ở điểm nào.
