---
title: "Scripts - Module 10"
module: 10
tags: [docker, sysops-infra, module-10, scripts]
---

# Scripts - Module 10: Docker Images

Script hỗ trợ build/audit image, dùng kèm [[../04-Commands|04-Commands.md]].

## Script gợi ý tự viết

| Script | Mục đích |
|---|---|
| `image-size-report.sh` | Liệt kê toàn bộ image cục bộ kèm kích thước, sắp xếp giảm dần — dùng định kỳ rà soát image nặng bất thường |
| `dockerfile-lint.sh` | Kiểm tra nhanh Dockerfile có `USER` non-root, có pin version ở `FROM`, có `.dockerignore` đi kèm hay không (kiểm tra bằng grep/regex đơn giản, không thay thế công cụ lint chuyên dụng như Hadolint) |
| `secret-leak-scan.sh` | Chạy `docker history --no-trunc` trên một image, grep các từ khóa nghi ngờ (`token`, `password`, `key`, `secret`) để phát hiện sớm secret bị lộ qua ARG/ENV theo Sự cố 5 ở [[../06-Troubleshooting|06-Troubleshooting.md]] |

## Quy ước

- Script bash, `set -euo pipefail`.
- Không script nào tự động `docker push` — mọi thao tác đẩy image lên registry phải qua bước xác nhận thủ công hoặc pipeline CI có kiểm soát.
- `secret-leak-scan.sh` chỉ mang tính hỗ trợ phát hiện nhanh cục bộ, không thay thế công cụ image scanning chuyên dụng (Trivy, Grype...) dùng trong pipeline CI/CD thật.
