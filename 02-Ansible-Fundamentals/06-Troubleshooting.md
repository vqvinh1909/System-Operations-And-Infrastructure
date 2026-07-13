---
title: "06 - Troubleshooting"
module: 2
tags: [ansible, sysops-infra, module-02, troubleshooting]
---

# 06 — Troubleshooting

## Lỗi 1: `UNREACHABLE! => ... Failed to connect to the host via ssh`

**Triệu chứng:** ad-hoc command hoặc playbook báo `UNREACHABLE` cho một hoặc nhiều host.

**Nguyên nhân thường gặp và cách xử lý theo thứ tự kiểm tra:**

```bash
# 1. Test SSH thu cong truoc, tach biet loi Ansible voi loi ha tang
ssh -i ~/.ssh/id_automation deploy@10.0.1.11

# 2. Neu SSH thu cong loi, day la loi ha tang (da xu ly o Module 01), khong phai loi Ansible
# 3. Neu SSH thu cong OK nhung Ansible van UNREACHABLE, kiem tra bien inventory
ansible-inventory -i inventory.yml --host web-01
# Xac nhan ansible_host, ansible_user, ansible_ssh_private_key_file dung chua

# 4. Chay voi -vvv de xem chi tiet lenh SSH Ansible thuc su goi
ansible web-01 -m ansible.builtin.ping -vvv
```

Lỗi phổ biến nhất trong nhóm này: quên khai báo `ansible_user` trong inventory khi user SSH khác với user hiện tại đang chạy Ansible, hoặc đường dẫn `ansible_ssh_private_key_file` sai.

## Lỗi 2: Inventory parse sai — host bị thiếu hoặc gán nhầm nhóm

**Triệu chứng:** `ansible-inventory --graph` không hiển thị đúng cấu trúc mong đợi.

**Nguyên nhân:** lỗi thụt lề YAML (đã học ở Module 01), hoặc nhầm `:children` (chỉ nhóm được chứa nhóm con) với `:vars` (chứa biến) trong cú pháp INI.

```bash
# Luon xac nhan bang --graph truoc khi tin tuong inventory dung
ansible-inventory -i inventory.yml --graph

# Neu nghi ngo, xuat toan bo JSON de doi chieu tung host
ansible-inventory -i inventory.yml --list
```

## Lỗi 3: `MODULE FAILURE` — "Could not find Python interpreter"

**Triệu chứng:** module báo lỗi liên quan `python interpreter` không tìm thấy trên managed node, dù SSH kết nối bình thường.

**Nguyên nhân:** managed node có nhiều bản Python (hoặc không có `python` symlink, chỉ có `python3`), Ansible không tự đoán đúng đường dẫn `python` mặc định.

**Cách xử lý:**
```bash
# Kiem tra python thuc te co tren managed node
ssh deploy@10.0.1.11 "which python3"

# Chi dinh ro interpreter qua bien (dat trong inventory hoac group_vars)
ansible_python_interpreter=/usr/bin/python3
```
Từ Ansible 2.8 trở đi, `ansible_python_interpreter` mặc định là `auto` (tự dò tìm) — lỗi này ngày nay ít gặp hơn nhưng vẫn xảy ra với managed node cấu hình khác thường (nhiều bản Python trong `venv`, container tối giản không có `/usr/bin/python3`).

## Lỗi 4: Ad-hoc command chạy lại vẫn báo `changed=1` thay vì `changed=0`

**Triệu chứng:** chạy cùng một ad-hoc command 2 lần liên tiếp, kỳ vọng lần 2 idempotent (`changed=0`) nhưng vẫn báo thay đổi.

**Nguyên nhân gần như luôn là:** dùng `command`/`shell` thay vì module chuyên biệt. Hai module này **không có khái niệm idempotency built-in** — chúng chạy lệnh vô điều kiện mỗi lần, không tự kiểm tra trạng thái trước.

**Cách xử lý:** thay `ansible all -m command -a "useradd alice"` bằng `ansible all -b -m ansible.builtin.user -a "name=alice state=present"` — module `user` tự kiểm tra user đã tồn tại chưa trước khi hành động.

## Lỗi 5: `ansible.cfg` không được áp dụng như mong đợi

**Triệu chứng:** đã sửa `ansible.cfg` nhưng hành vi Ansible không đổi.

**Nguyên nhân:** đang chạy lệnh từ một thư mục khác (không phải thư mục chứa `ansible.cfg` bạn vừa sửa), hoặc có biến môi trường `ANSIBLE_CONFIG` đang trỏ tới một file khác có độ ưu tiên cao hơn.

**Cách xử lý:**
```bash
# Xac nhan file cau hinh nao dang duoc doc
ansible-config view

# Kiem tra bien moi truong co dang ghi de khong
echo $ANSIBLE_CONFIG
```

## Lỗi 6: `WARNING: Platform ... on host ... is using the discovered Python interpreter`

**Triệu chứng:** warning xuất hiện khi chạy, không phải lỗi cứng nhưng gây nhiễu log.

**Nguyên nhân:** Ansible tự dò Python thành công nhưng cảnh báo rằng kết quả dò tìm này có thể thay đổi giữa các lần chạy nếu môi trường managed node thay đổi (ví dụ nâng cấp OS).

**Cách xử lý:** khai báo tường minh `ansible_python_interpreter` trong `group_vars/all.yml` cho môi trường production để loại bỏ tính không chắc chắn, đúng tinh thần "explicit tốt hơn implicit" trong vận hành doanh nghiệp.
