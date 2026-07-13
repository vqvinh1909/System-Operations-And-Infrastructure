---
title: "04 - Commands"
module: 16
tags: [docker, sysops-infra, module-16, registry, harbor, commands]
---

# 04 — Lệnh, Cấu hình, Ví dụ mẫu

## Dựng Private Registry đơn giản bằng `registry:2`

```bash
# Chạy registry cơ bản, không TLS, không auth - CHI dung trong lab/mang noi bo tin cay
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2
```

- `-v registry-data:/var/lib/registry`: bắt buộc phải mount volume — nếu không, mọi image sẽ mất khi container bị xóa (`docker rm`), vì `/var/lib/registry` bên trong container là nơi Distribution lưu blob/manifest.
- `--restart=always`: registry là hạ tầng nền tảng, cần tự khởi động lại sau khi Docker daemon restart hoặc server reboot.

```bash
# Gan tag va push mot image len registry noi bo
docker pull nginx:1.27
docker tag nginx:1.27 localhost:5000/nginx:1.27
docker push localhost:5000/nginx:1.27

# Pull lai tu registry noi bo de kiem chung
docker pull localhost:5000/nginx:1.27

# Liet ke danh sach repository dang co trong registry (goi thang OCI API)
curl http://localhost:5000/v2/_catalog

# Liet ke danh sach tag cua mot repository cu the
curl http://localhost:5000/v2/nginx/tags/list
```

## Cấu hình TLS cho Registry (bắt buộc khi dùng ngoài localhost)

Docker daemon **mặc định từ chối push/pull tới bất kỳ registry nào không dùng HTTPS**, trừ khi registry đó được khai báo tường minh là "insecure registry". Đây không phải giới hạn tùy chọn — là một quyết định thiết kế bảo mật cố ý.

```bash
# Cach 1 (chi dung lab/noi bo, KHONG dung production): khai bao insecure registry
# Sua file /etc/docker/daemon.json tren MOI client can push/pull
cat /etc/docker/daemon.json
# {
#   "insecure-registries": ["registry.company.local:5000"]
# }
sudo systemctl restart docker
```

```bash
# Cach 2 (dung production): tao certificate va chay registry voi TLS that
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout certs/domain.key -x509 -days 365 \
  -out certs/domain.crt \
  -subj "/CN=registry.company.local"

docker run -d \
  --name registry \
  --restart=always \
  -p 443:443 \
  -v registry-data:/var/lib/registry \
  -v "$(pwd)"/certs:/certs \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

## Thêm Basic Authentication cho `registry:2`

```bash
# Tao file mat khau bang htpasswd (thuong co san trong goi apache2-utils/httpd-tools)
mkdir -p auth
htpasswd -Bbn adminuser 'MatKhauManh!23' > auth/htpasswd

docker run -d \
  --name registry \
  --restart=always \
  -p 443:443 \
  -v registry-data:/var/lib/registry \
  -v "$(pwd)"/certs:/certs \
  -v "$(pwd)"/auth:/auth \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2

# Dang nhap tu client
docker login registry.company.local
```

## Cài đặt Harbor (offline installer — cách phổ biến nhất trong doanh nghiệp)

```bash
# 1. Tai bo cai dat offline tu GitHub release chinh thuc cua Harbor
wget https://github.com/goharbor/harbor/releases/download/v2.15.0/harbor-offline-installer-v2.15.0.tgz
tar xzf harbor-offline-installer-v2.15.0.tgz
cd harbor

# 2. Sua file cau hinh - BAT BUOC doi hostname va mat khau mac dinh
cp harbor.yml.tmpl harbor.yml
# Trong harbor.yml, chinh toi thieu:
#   hostname: registry.company.local
#   https.certificate: /path/to/cert.pem
#   https.private_key: /path/to/key.pem
#   harbor_admin_password: <mat khau manh, KHONG giu mac dinh Harbor12345>

# 3. Chay script cai dat - tu dong pull image Harbor va dung docker compose
sudo ./install.sh --with-trivy

# 4. Kiem tra cac container Harbor da chay
docker compose ps
```

> [!note] `--with-trivy`
> Cờ này bật tích hợp scanner Trivy ngay từ lúc cài đặt — không có cờ này, Harbor vẫn cài được nhưng thiếu khả năng scan lỗ hổng, phải cấu hình scanner riêng sau.

## Thao tác Harbor qua CLI (sau khi login)

```bash
# Login vao Harbor tu Docker client (giong het login vao registry:2 co auth)
docker login registry.company.local

# Tao project moi qua API (thay vi UI) - huu ich khi tu dong hoa bang script/Ansible
curl -u admin:'MatKhauManh!23' -X POST \
  "https://registry.company.local/api/v2.0/projects" \
  -H "Content-Type: application/json" \
  -d '{"project_name": "payment-team", "public": false}'

# Push image vao project vua tao
docker tag payment-service:v2.3 registry.company.local/payment-team/payment-service:v2.3
docker push registry.company.local/payment-team/payment-service:v2.3

# Kich hoat scan mot image cu the qua API
curl -u admin:'MatKhauManh!23' -X POST \
  "https://registry.company.local/api/v2.0/projects/payment-team/repositories/payment-service/artifacts/v2.3/scan"

# Xem ket qua scan qua API
curl -u admin:'MatKhauManh!23' \
  "https://registry.company.local/api/v2.0/projects/payment-team/repositories/payment-service/artifacts/v2.3/additions/vulnerabilities"
```

## Scan image bằng Docker Scout (không cần Harbor)

```bash
# Docker Scout tich hop san trong Docker CLI tu ban gan day, dung nhanh de kiem tra 1 image
docker scout quickview nginx:1.27
docker scout cves nginx:1.27
```

## Garbage Collection (dọn blob mồ côi) trên Harbor

```bash
# Qua UI: Administration > Garbage Collection > GC Now
# Qua API:
curl -u admin:'MatKhauManh!23' -X POST \
  "https://registry.company.local/api/v2.0/system/gc/schedule" \
  -H "Content-Type: application/json" \
  -d '{"schedule": {"type": "Manual"}}'
```

> [!warning] GC cần tạm ngừng push trong lúc chạy
> Garbage Collection trên registry Distribution (kể cả trong Harbor) không an toàn 100% để chạy đồng thời với các lệnh `push` khác — Harbor từ bản 2.x đã hỗ trợ "GC without downtime" bằng cơ chế khóa tạm thời, nhưng thực hành an toàn nhất trong doanh nghiệp vẫn là lên lịch GC vào khung giờ thấp điểm (ví dụ 2-4h sáng).
