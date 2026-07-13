---
title: "03 - Architecture"
module: 5
tags: [ansible, sysops-infra, module-05, architecture, diagram, vault, precedence]
---

# 03 — Kiến trúc & Sơ đồ

## Sơ đồ 1: Variable Precedence — chồng lớp từ thấp tới cao

```
+------------------------------------------------+
|    22. extra-vars (--extra-vars) - CAO NHAT    |
+------------------------------------------------+
+------------------------------------------------+
| 15-21. Role vars, block/task vars, set_fact... |
+------------------------------------------------+
+------------------------------------------------+
|       11-14. Facts, play vars/vars_files       |
+------------------------------------------------+
+------------------------------------------------+
|          3-10. group_vars, host_vars           |
+------------------------------------------------+
+------------------------------------------------+
|          2. Role defaults - THAP NHAT          |
+------------------------------------------------+
```

Đọc sơ đồ: hình dung như các lớp sơn phủ chồng lên nhau — lớp trên cùng (`extra-vars`) luôn là màu nhìn thấy cuối cùng, bất kể 4 lớp dưới có giá trị gì. Ngược lại, `Role defaults` là lớp nền, chỉ "lộ ra" khi không có bất kỳ lớp nào phía trên định nghĩa cùng biến đó. Bảng đầy đủ 22 cấp nằm ở [[02-Theory|02 - Theory]] mục 1 — sơ đồ này chỉ minh họa nguyên tắc chồng lớp, không thay thế việc tra bảng khi cần độ chính xác tuyệt đối.

## Sơ đồ 2: Luồng mã hóa và giải mã Ansible Vault

```
+----------------------------------------------+
|  File .yml plaintext (db_password: S3cret)   |
+----------------------------------------------+
                       |
                       v
+----------------------------------------------+
|            ansible-vault encrypt             |
+----------------------------------------------+
                       |
                       v
+----------------------------------------------+
| File .yml ma hoa AES256 ($ANSIBLE_VAULT;1.1) |
+----------------------------------------------+
                       |
                       v
+----------------------------------------------+
|         git commit + push - AN TOAN          |
+----------------------------------------------+
                       |
                       v
+----------------------------------------------+
|    ansible-playbook chay, can --vault-id     |
+----------------------------------------------+
                       |
                       v
+----------------------------------------------+
|  Ansible tu giai ma trong RAM luc chay Task  |
+----------------------------------------------+
```

Đọc sơ đồ: nội dung chỉ tồn tại ở dạng plaintext tại 2 thời điểm — lúc kỹ sư soạn thảo ban đầu (trước bước `encrypt`), và tạm thời trong RAM của Control Node khi Ansible giải mã để dùng lúc chạy Task. Ở mọi thời điểm khác — trên đĩa, trong Git, trong lịch sử commit — nội dung luôn ở dạng mã hóa AES256. Đây là lý do password Vault (bước `--vault-id`) là **mắt xích bảo mật duy nhất** — mất password Vault đồng nghĩa mất toàn bộ secret được bảo vệ bởi nó.
