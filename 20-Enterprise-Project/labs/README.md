# Labs — Module 20: Enterprise Project

Thư mục này lưu **toàn bộ mã nguồn thật** của đồ án cuối khóa mô tả ở [[../05-Labs|05-Labs.md]] — đây là thư mục quan trọng nhất trong toàn bộ khóa học, vì đây chính là dự án bạn sẽ đưa vào portfolio cá nhân.

Gợi ý cấu trúc (phản ánh đúng một dự án Ansible + Docker thật, không phải bài lab đơn lẻ):

```
labs/
├── inventory.ini
├── ansible.cfg
├── site.yml
├── roles/
│   ├── common/
│   ├── docker/
│   ├── haproxy/
│   │   ├── templates/haproxy.cfg.j2
│   │   └── tasks/main.yml
│   ├── webapp/
│   │   ├── files/docker-compose.yml
│   │   └── tasks/main.yml
│   └── database/
│       ├── files/docker-compose.yml
│       └── tasks/main.yml
├── monitoring/
│   ├── prometheus.yml
│   └── grafana-dashboard-export.json
├── backup/
│   ├── backup-mysql.sh
│   ├── backup-rsync.sh
│   └── crontab-backup.txt
└── notes.md
```

Trong `notes.md`, ghi lại: quyết định kiến trúc bạn tự đưa ra (và lý do), toàn bộ 20 tình huống ở [[../06-Troubleshooting|06-Troubleshooting.md]] bạn đã tự tái tạo và xử lý trên hạ tầng thật, cùng RPO/RTO bạn tự đặt ra cho dự án. Đây là tài liệu bạn sẽ dùng lại khi kể về dự án này trong phỏng vấn xin việc.

> [!warning] Không commit bí mật thật
> Nếu bạn đẩy thư mục này lên một Git repository công khai (ví dụ để làm portfolio), tuyệt đối không commit mật khẩu thật, private key SSH thật, hay bất kỳ giá trị nhạy cảm nào ở dạng plaintext — dùng Ansible Vault (đã học ở Module 05) để mã hóa mọi file chứa bí mật trước khi commit.
