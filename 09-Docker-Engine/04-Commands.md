---
title: "04 - Commands"
module: 9
tags: [docker, sysops-infra, module-09, commands, reference]
---

# 04 - Lệnh, cú pháp, ví dụ cấu hình

## 1. Kiểm tra kiến trúc/thông tin daemon

```bash
# Xem thông tin tổng quan daemon: storage driver, cgroup driver, số container/image...
docker info

# Xem phiên bản Client và Server (Engine) riêng biệt - hai con số này co the khac nhau
docker version

# Xem daemon co dang chay khong, do systemd quan ly
systemctl status docker

# Xem log realtime cua daemon (rat quan trong khi troubleshoot)
journalctl -u docker.service -f

# Xem socket dang duoc dockerd lang nghe
ss -lx | grep docker
```

## 2. daemon.json — cấu hình daemon

Vị trí mặc định: `/etc/docker/daemon.json`. File này ở định dạng JSON thuần, không có comment. Sau khi sửa, chạy `sudo systemctl restart docker` (một số option hỗ trợ `reload`, đa số option quan trọng như storage-driver cần restart hẳn).

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://registry-mirror.company.local"],
  "insecure-registries": [],
  "default-address-pools": [
    { "base": "172.30.0.0/16", "size": 24 }
  ],
  "live-restore": true,
  "userland-proxy": false
}
```

**Giải thích từng key quan trọng (vì sao cấu hình, không chỉ "là gì"):**

| Key | Vì sao cần |
|---|---|
| `log-driver` + `log-opts` | Mặc định Docker không giới hạn dung lượng log của container — một container log liên tục có thể lấp đầy ổ đĩa `/var/lib/docker`. Giới hạn `max-size`/`max-file` là thực hành bắt buộc trên production. |
| `storage-driver: overlay2` | Chỉ định rõ storage driver (liên hệ Module 08 OverlayFS) thay vì để Docker tự chọn — tránh trường hợp daemon chọn driver kém hiệu năng hơn trên một số distro cũ. |
| `exec-opts: native.cgroupdriver=systemd` | Khớp cgroup driver của Docker với cgroup driver mà hệ thống init (systemd) đang dùng để quản lý cgroup. Không khớp driver (`cgroupfs` vs `systemd`) là nguyên nhân phổ biến gây lỗi khó hiểu khi giới hạn tài nguyên không áp dụng đúng, đặc biệt khi node đó sau này tham gia cụm Kubernetes (kubelet cũng yêu cầu khớp cgroup driver). |
| `registry-mirrors` | Trỏ tới registry mirror nội bộ để giảm phụ thuộc Docker Hub và tránh rate-limit khi kéo image liên tục từ CI/CD. |
| `live-restore` | Cho phép container tiếp tục chạy khi `dockerd` restart hoặc bị dừng (không phải khi reboot cả server) — tận dụng đúng kiến trúc shim đã học ở [[02-Theory]]. |
| `userland-proxy: false` | Tắt cơ chế proxy userspace cho port mapping, để NAT xử lý hoàn toàn bằng iptables/netfilter ở kernel — hiệu năng tốt hơn, ít tiến trình phụ hơn. |

## 3. Vòng đời container

```bash
# Chi tao container, chua chay (trang thai CREATED)
docker create --name web nginx:1.27-alpine

# Chay container da tao
docker start web

# Tao + chay ngay lap tuc (tuong duong create + start), -d = detached (chay nen)
docker run -d --name web -p 8080:80 nginx:1.27-alpine

# Xem danh sach container dang chay
docker ps

# Xem TAT CA container ke ca da exited
docker ps -a

# Xem chi tiet trang thai, cau hinh, network cua container (JSON)
docker inspect web

# Chi lay truong State (trang thai) de doc nhanh
docker inspect web --format '{{json .State}}'

# Dung container: gui SIGTERM, cho 10s (mac dinh), roi SIGKILL neu chua thoat
docker stop web

# Dung voi grace period tuy chinh (vi du 30 giay cho ung dung dong ket noi DB)
docker stop -t 30 web

# Buoc dung ngay lap tuc bang SIGKILL, khong grace period
docker kill web

# Gui signal khac thay vi SIGKILL mac dinh (vi du SIGHUP de reload config)
docker kill --signal=SIGHUP web

# Tam dung (freeze) tien trinh ben trong, khong gui signal
docker pause web

# Giai dong, tiep tuc chay tu diem bi dong bang
docker unpause web

# Khoi dong lai (tuong duong stop roi start)
docker restart web

# Xoa container da dung (that bai neu container dang chay)
docker rm web

# Buoc xoa ke ca dang chay (tu dong kill truoc khi xoa)
docker rm -f web

# Xoa tat ca container da exited de don dep
docker container prune
```

## 4. Theo dõi và debug container

```bash
# Xem log cua container
docker logs web

# Theo doi log realtime, giong tail -f
docker logs -f web

# Xem 100 dong log gan nhat
docker logs --tail 100 web

# Chay lenh moi ben trong container dang chay (mo shell de debug)
docker exec -it web /bin/sh

# Xem tien trinh dang chay ben trong container (tu host, khong can vao container)
docker top web

# Xem thong ke tai nguyen realtime: CPU, memory, network I/O (giong lenh top nhung cho container)
docker stats web

# Xem exit code cua lan chay gan nhat
docker inspect web --format '{{.State.ExitCode}}'
```

## 5. Registry, image naming

```bash
# Dang nhap Docker Hub (hoac registry noi bo neu chi dinh)
docker login registry.company.local:5000

# Pull image, dang day du: registry/repository:tag
docker pull registry.company.local:5000/backend-team/payment-api:v2.3.1

# Pull tu Docker Hub (khong can ghi registry vi mac dinh la docker.io)
docker pull nginx:1.27-alpine

# Gan tag moi cho mot image da co (khong tao ban sao du lieu, chi tao them reference)
docker tag payment-api:v2.3.1 registry.company.local:5000/backend-team/payment-api:v2.3.1

# Day image len registry
docker push registry.company.local:5000/backend-team/payment-api:v2.3.1

# Xem danh sach image dang co cuc bo
docker images

# Xem Image ID day du (SHA-256 digest)
docker images --no-trunc

# Xoa image (that bai neu con container dang dung image do)
docker rmi nginx:1.27-alpine
```

## 6. Quyền và socket

```bash
# Xem quyen file socket
ls -l /var/run/docker.sock
# vi du output: srw-rw---- 1 root docker 0 Jul 12 08:00 /var/run/docker.sock

# Xem user hien tai co thuoc nhom docker khong
groups $USER

# Them user vao nhom docker (CAN CAN NHAC KY - xem 02-Theory muc 4 va 06-Troubleshooting)
sudo usermod -aG docker vinh
# Luu y: user can logout/login lai (hoac newgrp docker) de nhom moi co hieu luc

# Cach an toan hon de test lenh docker ma khong doi quyen han vinh vien: dung sudo cho tung lenh
sudo docker ps
```

## 7. Ví dụ chạy container với giới hạn tài nguyên (liên hệ cgroup Module 08)

```bash
# Gioi han 512MB RAM va 0.5 CPU core
docker run -d --name app \
  --memory=512m \
  --cpus=0.5 \
  myapp:latest

# Chi dinh restart policy - tu dong khoi dong lai neu container thoat khong mong muon
docker run -d --name app \
  --restart=unless-stopped \
  myapp:latest
```

---

Tiếp theo: [[05-Labs]] để thực hành trực tiếp các lệnh này.
