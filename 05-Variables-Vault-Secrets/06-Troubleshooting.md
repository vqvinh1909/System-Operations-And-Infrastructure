---
title: "06 - Troubleshooting"
module: 5
tags: [ansible, sysops-infra, module-05, troubleshooting, vault]
---

# 06 — Troubleshooting

## Lỗi 1: Biến ra giá trị "sai" dù đã khai báo đúng ở nơi tưởng là đúng

**Triệu chứng:** bạn sửa `group_vars/webservers.yml` nhưng giá trị chạy thực tế không đổi.

**Nguyên nhân gần như luôn là:** một nguồn có độ ưu tiên **cao hơn** đang ghi đè — phổ biến nhất là `host_vars` cho riêng host đó, hoặc một Task/Role có `vars:` cục bộ, hoặc `--extra-vars` còn sót lại trong alias/script CI mà bạn quên.

**Cách xử lý:**
```bash
# Xem gia tri cuoi cung da resolve cho dung host do
ansible-inventory -i inventory.yml --host web-01

# Tim toan bo noi bien nay duoc khai bao trong repo
grep -rn "ten_bien" group_vars/ host_vars/ roles/ playbooks/
```
Nguyên tắc debug: đi từ nguồn ưu tiên **cao nhất** xuống thấp dần (extra-vars → task/block vars → role vars → play vars → host_vars → group_vars → role defaults) thay vì đoán mò — bảng đầy đủ ở [[02-Theory|02 - Theory]] mục 1.

## Lỗi 2: `ERROR! Decryption failed`

**Triệu chứng:** `ansible-playbook` hoặc `ansible-vault view/edit` báo lỗi giải mã thất bại.

**Nguyên nhân thường gặp:**
- Sai password Vault (gõ nhầm khi `--ask-vault-pass`, hoặc file password đã bị thay đổi/rotate mà chưa cập nhật).
- File Vault được mã hóa bằng Vault ID khác với Vault ID đang cung cấp lúc chạy (mỗi Vault ID có password riêng, không dùng lẫn được).
- File bị chỉnh sửa thủ công ngoài `ansible-vault edit` (ví dụ mở bằng editor thường, gõ nhầm ký tự vào nội dung mã hóa) — làm hỏng cấu trúc mã hóa.

**Cách xử lý:** xác nhận đúng file password/Vault ID đang dùng khớp với file đã mã hóa (`head -1 file.yml` xem dòng `$ANSIBLE_VAULT;1.1;AES256` để xác nhận file thực sự đã mã hóa, không phải file thường bị lỗi khác). Nếu nghi ngờ file bị hỏng cấu trúc, khôi phục từ bản Git commit trước đó.

## Lỗi 3: Vô tình commit plaintext secret lên Git

**Triệu chứng:** phát hiện một file chứa mật khẩu/API key dạng plaintext đã bị `git push` lên remote (kể cả private repo).

**Xử lý khẩn cấp theo thứ tự:**
1. **Coi secret đó là đã lộ ngay lập tức** — đổi mật khẩu/API key thật trên hệ thống liên quan, không chờ dọn dẹp Git xong mới đổi.
2. Xóa file khỏi lịch sử Git bằng công cụ chuyên dụng (`git filter-repo` hoặc BFG Repo-Cleaner) — chỉ xóa ở commit mới nhất là **không đủ**, lịch sử cũ vẫn còn.
3. Force-push sau khi dọn lịch sử, thông báo toàn bộ team clone lại repo (lịch sử đã bị viết lại).
4. Mã hóa lại giá trị mới bằng Ansible Vault trước khi commit lại.

**Phòng ngừa:** thêm `git-secrets` hoặc `pre-commit hook` quét pattern secret phổ biến (AWS key, private key header) trước khi cho phép commit.

## Lỗi 4: `--vault-id` không tìm đúng password cho đúng file

**Triệu chứng:** Playbook dùng nhiều file Vault mã hóa bởi các Vault ID khác nhau, chạy báo lỗi giải mã dù password đúng cho từng file riêng lẻ.

**Nguyên nhân:** Ansible cần **tất cả** Vault ID liên quan được cung cấp cùng lúc trong một lệnh `ansible-playbook` — thiếu một Vault ID nào đó khiến file tương ứng không giải mã được (dù các file khác vẫn ổn).

**Cách xử lý:** khai báo `vault_identity_list` trong `ansible.cfg` (mục 4, [[04-Commands|04 - Commands]]) để không phải nhớ liệt kê đủ mọi `--vault-id` mỗi lần chạy tay.

## Lỗi 5: `encrypt_string` output dán vào YAML bị lỗi thụt lề

**Triệu chứng:** sau khi dán kết quả `ansible-vault encrypt_string` vào file, YAML báo lỗi cú pháp.

**Nguyên nhân:** output của `encrypt_string` là khối nhiều dòng, cần thụt lề **nhất quán** dưới key biến, thường bị lệch khi copy-paste qua nhiều tầng editor/terminal khác nhau.

**Cách xử lý:** dán nguyên khối output (không sửa tay từng dòng), dùng `python3 -c "import yaml; yaml.safe_load(open('file.yml'))"` (đã học ở Module 01) để xác nhận cú pháp hợp lệ trước khi commit.
