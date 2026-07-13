---
title: "02 - Theory"
module: 5
tags: [ansible, sysops-infra, module-05, variable-precedence, vault, secrets]
---

# 02 — Lý thuyết

## 1. Variable Precedence — 22 cấp, từ thấp tới cao

Khi cùng một biến được khai báo ở nhiều nơi, Ansible áp dụng **giá trị từ nguồn có độ ưu tiên cao nhất**, theo thứ tự cố định sau (thấp nhất tới cao nhất):

| # | Nguồn | Ghi chú |
|---|---|---|
| 1 | Giá trị dòng lệnh (`-u`, không phải `-e`) | Thấp nhất, không phải "biến" theo nghĩa đầy đủ |
| 2 | Role defaults (`roles/x/defaults/main.yml`) | Thấp nhất trong các biến thật — thiết kế để bị ghi đè |
| 3 | Inventory file/script — group vars | |
| 4 | Inventory `group_vars/all` | |
| 5 | Playbook `group_vars/all` | |
| 6 | Inventory `group_vars/*` (nhóm cụ thể) | |
| 7 | Playbook `group_vars/*` (nhóm cụ thể) | |
| 8 | Inventory file/script — host vars | |
| 9 | Inventory `host_vars/*` | |
| 10 | Playbook `host_vars/*` | |
| 11 | Facts / cached `set_facts` | Dữ liệu Ansible tự thu thập (Module 02) |
| 12 | Play `vars:` | |
| 13 | Play `vars_prompt:` | |
| 14 | Play `vars_files:` | |
| 15 | Role `vars:` (`roles/x/vars/main.yml`) | Cao — không nên bị ghi đè tùy tiện |
| 16 | Block `vars:` | Chỉ áp dụng cho Task trong block đó |
| 17 | Task `vars:` | Chỉ áp dụng cho Task đó |
| 18 | `include_vars` | |
| 19 | `set_fact` / biến `register` | |
| 20 | Role params (khi gọi `role`/`include_role` kèm tham số) | |
| 21 | Include params | |
| 22 | Extra vars (`--extra-vars`/`-e`) | **Cao nhất tuyệt đối** — không gì ghi đè được |

**Hai nguyên tắc thực dụng cần nhớ (không cần thuộc lòng cả 22 dòng):**
- **`--extra-vars` luôn thắng.** Đây là lý do CI/CD pipeline thường dùng `--extra-vars` để truyền giá trị theo môi trường (`env=production`) — đảm bảo không bị bất kỳ file cấu hình nào trong repo vô tình ghi đè.
- **Role defaults luôn thua.** Đây là lý do `roles/x/defaults/main.yml` là nơi đúng để đặt giá trị mặc định "hợp lý" cho một Role dùng chung — người dùng Role luôn có thể ghi đè dễ dàng qua bất kỳ nguồn nào khác (group_vars, host_vars, extra-vars...).

## 2. Ansible Vault — mã hóa dữ liệu nhạy cảm

Ansible Vault mã hóa nội dung bằng thuật toán **AES256** (tại thời điểm biên soạn, đây vẫn là cipher duy nhất được Ansible chính thức hỗ trợ). Có 2 cách dùng chính:

### 2.1 Mã hóa toàn bộ file

```bash
# Tao file moi, da duoc ma hoa ngay tu dau
ansible-vault create group_vars/production/vault.yml

# Ma hoa mot file YAML thuong da co san
ansible-vault encrypt group_vars/production/secrets.yml

# Sua noi dung file da ma hoa (tu dong giai ma - sua - ma hoa lai)
ansible-vault edit group_vars/production/vault.yml

# Xem noi dung ma khong sua
ansible-vault view group_vars/production/vault.yml

# Giai ma vinh vien (thuong chi dung khi migrate, khong khuyen nghi de plaintext lau)
ansible-vault decrypt group_vars/production/vault.yml
```

File sau khi mã hóa có nội dung dạng:
```
$ANSIBLE_VAULT;1.1;AES256
66386439653236336462626566653063336164663966303231363934653561363964363833316234...
```

Có thể **commit file này lên Git một cách an toàn** — không ai đọc được nội dung thật nếu không có password Vault.

### 2.2 Mã hóa một biến đơn lẻ (`encrypt_string`)

Khi chỉ 1-2 biến nhạy cảm trong một file phần lớn là plaintext (dễ đọc, dễ review hơn mã hóa cả file):

```bash
ansible-vault encrypt_string 'S3cretPassword123' --name 'db_password'
```

Kết quả dán trực tiếp vào file YAML thường (không mã hóa):
```yaml
db_user: appuser
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653236336462626566653063336164663966303231363934653561...
db_host: 10.0.1.20
```

Ansible tự nhận diện tag `!vault` và giải mã đúng biến đó khi chạy, các biến khác trong cùng file vẫn đọc được bình thường — cân bằng giữa bảo mật và khả năng review code trong Pull Request.

### 2.3 Multiple Vault ID — nhiều password cho nhiều môi trường

Doanh nghiệp thường không dùng chung 1 password Vault cho mọi môi trường (rủi ro: người có quyền đọc secret dev cũng đọc được secret production). **Vault ID** cho phép gắn nhãn (label) cho từng nguồn password:

```bash
# Ma hoa file rieng cho moi truong dev, gan nhan "dev"
ansible-vault encrypt --vault-id dev@prompt group_vars/dev/vault.yml

# Ma hoa file rieng cho moi truong production, gan nhan "prod", doc password tu file
ansible-vault encrypt --vault-id prod@~/.vault_pass_prod.txt group_vars/production/vault.yml

# Chay playbook can ca 2 vault id cung luc
ansible-playbook site.yml --vault-id dev@prompt --vault-id prod@~/.vault_pass_prod.txt
```

Cú pháp `ID@SOURCE`: `ID` là nhãn tùy chọn, `SOURCE` là nơi lấy password — có thể là `prompt` (hỏi tương tác), đường dẫn tới file chứa password, hoặc đường dẫn tới một script thực thi trả về password (tích hợp với secret manager bên ngoài như HashiCorp Vault, AWS Secrets Manager).

## 3. Tích hợp Vault vào Playbook — trong suốt với người dùng

Điểm quan trọng: **Task trong Playbook không cần biết biến có được mã hóa hay không.** Chỉ cần khai báo file Vault trong `vars_files` hoặc để đúng chỗ trong `group_vars`/`host_vars` (Ansible tự động load), Task tham chiếu biến bình thường:

```yaml
# group_vars/production/vault.yml (da ma hoa bang ansible-vault)
db_password: "S3cretPassword123"
```

```yaml
# Task su dung bien - khong can biet bien tu file ma hoa hay khong
- name: Cau hinh ket noi database
  ansible.builtin.template:
    src: db_config.j2
    dest: /etc/myapp/db.conf
  vars:
    password: "{{ db_password }}"
```

Khi chạy `ansible-playbook`, cần cung cấp password Vault để giải mã (qua `--ask-vault-pass`, `--vault-password-file`, hoặc `--vault-id`) — nếu thiếu, Ansible báo lỗi rõ ràng, không âm thầm chạy với giá trị rỗng.

## 4. Secrets Management — nguyên tắc thực hành doanh nghiệp

- **Không bao giờ commit plaintext secret**, kể cả trong lịch sử Git đã bị xóa ở commit sau — lịch sử Git vẫn giữ lại, cần công cụ như `git filter-repo` để xóa triệt để nếu lỡ commit nhầm (và vẫn nên coi secret đó là đã lộ, phải xoay vòng/đổi mới).
- **Password Vault không lưu trong repo**, không gửi qua chat/email — lưu trong password manager của team, hoặc tích hợp script Vault ID trỏ tới secret manager tập trung (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) cho môi trường production.
- **Tách biệt secret theo môi trường** (dev/staging/production dùng Vault ID và password khác nhau) — hạn chế phạm vi ảnh hưởng nếu một password Vault bị lộ.
- **Rotate (xoay vòng) định kỳ:** cả secret bên trong (`ansible-vault rekey` để đổi password Vault) lẫn giá trị secret thật (đổi mật khẩu database định kỳ, không chỉ đổi password mã hóa).
- **Giới hạn số người biết password Vault** theo nguyên tắc least privilege — không phải mọi kỹ sư trong team cần quyền giải mã secret production.
- **`.gitignore` file password Vault cục bộ** (`.vault_pass.txt`) — chỉ file `.yml` đã mã hóa mới được commit, file chứa password thật tuyệt đối không.
