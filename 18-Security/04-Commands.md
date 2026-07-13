---
title: "04 - Commands"
module: 18
tags: [docker, sysops-infra, module-18, security, commands]
---

# 04 — Lệnh, Cấu hình, Ví dụ mẫu

## Cài đặt Rootless Docker

```bash
# Cai dat goi ho tro rootless (tren distro dua tren apt)
sudo apt-get install -y docker-ce-rootless-extras uidmap

# Dam bao user hien tai co dai UID/GID rieng trong subuid/subgid
grep "$(whoami)" /etc/subuid /etc/subgid

# Dang xuat khoi phien dockerd rootful (neu dang chay) roi chay script cai dat rootless
# BANG CHINH TAI KHOAN USER THUONG (khong sudo)
dockerd-rootless-setuptool.sh install

# Tro bien moi truong ve socket rootless
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock

# Kiem tra dockerd rootless dang chay bang UID nao (phai la UID cua user thuong, khong phai 0)
docker info | grep -i rootless
ps -o user= -C dockerd
```

```bash
# Bat rootless Docker tu dong khoi dong cung user session
systemctl --user enable docker
sudo loginctl enable-linger $(whoami)   # cho phep chay ca khi user chua login (vd sau reboot)
```

## Capabilities — `--cap-drop` / `--cap-add`

```bash
# Nguyen tac chuan: bo het, chi cap lai dung thu can
docker run -d --name web \
  --cap-drop=ALL \
  nginx:1.27
# Voi nginx binh thuong (khong bind port <1024 tu ben trong, dung port map cua Docker)
# thuong KHONG can cap-add nao them.
```

```bash
# Vi du can 1 capability cu the: ung dung can bind thang port 80 BEN TRONG container
# (khong qua co che map port cua Docker) thi can lai CAP_NET_BIND_SERVICE
docker run -d --name web-bind-80 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  custom-app:v1
```

Bảng capability hay gặp và ý nghĩa thực tế:

| Capability | Cho phép làm gì | Khi nào thật sự cần |
|---|---|---|
| `NET_BIND_SERVICE` | Bind port < 1024 | App tự bind port thấp bên trong container, không qua port mapping |
| `NET_ADMIN` | Cấu hình interface mạng, firewall | Container đóng vai trò VPN/proxy/firewall thật sự |
| `SYS_ADMIN` | Rất rộng — mount, mở namespace... | Gần như không bao giờ cần cho app thông thường; tránh cấp |
| `CHOWN` | Đổi owner file | App cần tự đổi quyền file lúc chạy (hiếm) |
| `SETUID` / `SETGID` | Đổi UID/GID tiến trình | App tự hạ quyền nội bộ lúc khởi động (một số entrypoint script) |

```bash
# Kiem tra capability mac dinh Docker dang cap cho 1 container dang chay
docker inspect --format '{{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}}' web
```

## Seccomp

```bash
# Chay container voi seccomp profile MAC DINH cua Docker (khuyen nghi, khong can khai bao gi them)
docker run -d --name web nginx:1.27

# Chay voi mot seccomp profile TUY CHINH (file JSON tu viet)
docker run -d --name web \
  --security-opt seccomp=./custom-seccomp.json \
  nginx:1.27

# KHONG khuyen nghi (chi dung de test/debug tam thoi) - tat han seccomp
docker run -d --name web \
  --security-opt seccomp=unconfined \
  nginx:1.27
```

```json
// custom-seccomp.json - vi du RUT GON, minh hoa cau truc (khong dung nguyen ban production)
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "accept", "connect", "socket"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

> [!warning] Đừng tự viết seccomp profile từ đầu cho production
> Ví dụ trên chỉ minh họa cấu trúc file. Viết một seccomp profile đủ để một ứng dụng thật hoạt động bình thường nhưng vẫn chặt chẽ đòi hỏi liệt kê chính xác từng syscall cần thiết — làm thiếu sẽ khiến app lỗi khó hiểu (xem [[06-Troubleshooting]]), làm thừa sẽ mất tác dụng bảo mật. Thực hành phổ biến: bắt đầu từ seccomp profile mặc định của Docker, chỉ thu hẹp thêm khi có công cụ hỗ trợ sinh profile tự động dựa trên hành vi thật của ứng dụng (theo dõi syscall app thực sự gọi trong quá trình chạy thử).

## AppArmor

```bash
# Kiem tra AppArmor profile Docker mac dinh (docker-default) da load vao kernel chua
sudo aa-status | grep docker

# Chay container voi profile mac dinh (khong can khai bao - la mac dinh neu host co AppArmor)
docker run -d --name web nginx:1.27
docker inspect --format '{{.AppArmorProfile}}' web
# Ket qua thuong thay: docker-default

# Chay voi mot profile AppArmor TUY CHINH da load san vao kernel
docker run -d --name web \
  --security-opt apparmor=my-custom-profile \
  nginx:1.27

# Chay khong co AppArmor (KHONG khuyen nghi cho production)
docker run -d --name web \
  --security-opt apparmor=unconfined \
  nginx:1.27
```

## Image Scanning

```bash
# Docker Scout - tich hop san trong Docker CLI
docker scout quickview nginx:1.27
docker scout cves nginx:1.27
docker scout cves --only-severity critical,high nginx:1.27

# Trivy - cong cu quet ma nguon mo pho bien, dung doc lap khong can Harbor
trivy image nginx:1.27
trivy image --severity CRITICAL,HIGH nginx:1.27

# Dua ket qua scan thanh gate trong CI/CD - exit code khac 0 neu vuot nguong
trivy image --severity CRITICAL --exit-code 1 nginx:1.27
```

## Docker Bench for Security

```bash
# Clone va tu build - KHONG dung image dung san tren Docker Hub (da lỗi thoi)
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security

# Chay truc tiep bang script shell (khuyen nghi 2026)
sudo sh docker-bench-security.sh

# Chi chay mot nhom kiem tra cu the (vi du chi container runtime - phan 5 cua CIS Benchmark)
sudo sh docker-bench-security.sh -c container_runtime

# Xuat ket qua ra file de luu lai/so sanh qua thoi gian
sudo sh docker-bench-security.sh -l /var/log/docker-bench-$(date +%F).log
```

> [!note] Đọc kết quả Docker Bench
> Mỗi dòng bắt đầu bằng `[PASS]`, `[WARN]`, hoặc `[INFO]`. `[WARN]` không có nghĩa "phải sửa ngay lập tức" — nhiều mục mang tính khuyến nghị tùy ngữ cảnh (ví dụ khuyến nghị dùng Docker Swarm secrets chỉ áp dụng nếu bạn thật sự dùng Swarm). Ưu tiên đọc kỹ mô tả từng `[WARN]`, đối chiếu với hạ tầng thật của bạn, thay vì cố "làm sạch" toàn bộ output một cách máy móc.
