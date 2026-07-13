# Labs — Module 17: Logging & Monitoring

Thư mục này dùng để lưu toàn bộ code/config bạn tự viết khi làm theo [[../05-Labs|05-Labs.md]] của module này.

Gợi ý cấu trúc:

```
labs/
├── lab1-log-rotation/
│   └── notes.md
├── lab2-metrics-stack/
│   ├── docker-compose.yml
│   ├── prometheus.yml
│   └── notes.md
├── lab3-logs-stack/
│   ├── docker-compose.yml
│   ├── config.alloy
│   └── notes.md
└── lab4-dashboard-alert/
    ├── dashboard-export.json
    └── notes.md
```

Hãy lưu lại `notes.md` cho mỗi lab ghi chú lại: lỗi gặp phải, cách bạn tự sửa, và điều bạn hiểu thêm sau khi làm — đây là tài liệu ôn tập giá trị nhất khi bạn quay lại xem sau vài tháng. Với Lab 4, nên export dashboard Grafana ra JSON (`Dashboard settings → JSON Model`) để lưu lại cấu hình panel/alert đã tự tay xây dựng.
