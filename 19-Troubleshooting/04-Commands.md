---
title: "04 - Commands"
module: 19
tags: [docker, sysops-infra, module-19, troubleshooting, commands]
---

# 04 — Bộ lệnh chẩn đoán theo từng tầng

## Tầng Container

```bash
# Trang thai tong quan - cot STATUS phan biet Up / Exited / Restarting
docker ps -a

# Chi tiet toan bo cau hinh + trang thai runtime cua 1 container
docker inspect <container>

# Rieng exit code cua lan dung gan nhat
docker inspect --format '{{.State.ExitCode}}' <container>

# Rieng ly do OOMKilled co dung khong (true/false)
docker inspect --format '{{.State.OOMKilled}}' <container>

# So lan container da bi restart (theo restart policy)
docker inspect --format '{{.RestartCount}}' <container>

# Log gan nhat, kem timestamp - buoc dau tien luon lam
docker logs --tail 100 -t <container>

# Vao ben trong container dang chay de kiem tra truc tiep
docker exec -it <container> sh

# Xem timeline su kien cua Docker daemon (start, die, restart, oom...)
docker events --since 30m

# Trang thai healthcheck (neu image co khai bao HEALTHCHECK)
docker inspect --format '{{json .State.Health}}' <container> | python3 -m json.tool
```

## Tầng Network

```bash
# Danh sach network va container nao thuoc network nao
docker network ls
docker network inspect <network>

# Kiem tra DNS noi bo cua Docker tu ben trong container
docker exec -it <containerA> nslookup <containerB>
docker exec -it <containerA> ping -c 3 <containerB>

# Kiem tra port dang lang nghe BEN TRONG container
docker exec -it <container> ss -tlnp

# Kiem tra port mapping tu host
docker port <container>
sudo ss -tlnp | grep <port>

# Kiem tra NAT/iptables rule ma Docker daemon tu tao (port publish di qua day)
sudo iptables -t nat -L DOCKER -n -v

# Kiem tra bridge network va routing tren host
ip addr show docker0
ip route
brctl show 2>/dev/null || ip link show type bridge

# Bat goi tin de debug sau khi da loai tru cac buoc tren (dung khi that su can)
sudo tcpdump -i docker0 host <container-ip> -n
```

## Tầng Storage

```bash
# Dung luong tong the cua Docker (image, container, volume, build cache)
docker system df
docker system df -v      # chi tiet tung image/volume

# Danh sach volume va noi luu that tren host
docker volume ls
docker volume inspect <volume>

# Kiem tra UID/GID that su so huu 1 thu muc bind mount tren host
ls -lan /path/tren/host

# Kiem tra UID ma tien trinh BEN TRONG container dang chay bang
docker exec <container> id

# Don dep an toan (KHONG dung --volumes neu khong chac chan, se xoa ca du lieu)
docker system prune -a           # xoa image/container/network khong dung, GIU volume
docker system prune -a --volumes # xoa CA volume khong con container nao tham chieu - can trong

# Kiem tra dung luong that tren host noi Docker luu du lieu
df -h /var/lib/docker
sudo du -sh /var/lib/docker/overlay2/* 2>/dev/null | sort -rh | head -10
```

## Tầng Performance (Host/Kernel)

```bash
# Xem nhanh CPU/MEM/NET/IO cua tat ca container dang chay
docker stats --no-stream

# Doc truc tiep cgroup (v2 - kiem tra bang lenh duoi de biet dang dung ban nao)
stat -fc %T /sys/fs/cgroup/    # ket qua "cgroup2fs" nghia la dang dung cgroup v2

CID=$(docker inspect --format '{{.Id}}' <container>)
cat /sys/fs/cgroup/system.slice/docker-${CID}.scope/memory.current
cat /sys/fs/cgroup/system.slice/docker-${CID}.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-${CID}.scope/cpu.stat

# Tai nguyen TONG THE cua host - luon doi chieu voi so lieu tung container o tren
vmstat 1 5
iostat -x 1 5
top -o %CPU

# Kiem tra kernel co vua kich hoat OOM Killer khong, va giet tien trinh nao
dmesg -T | grep -i "killed process"
sudo journalctl -k | grep -i "out of memory"
```

## Bảng tra nhanh: "Tôi thấy X, tôi nên chạy lệnh gì"

| Triệu chứng | Lệnh nên chạy đầu tiên |
|---|---|
| Container `Exited` ngay | `docker logs`, `docker inspect --format '{{.State.ExitCode}}'` |
| Container cứ restart | `docker events`, `docker inspect --format '{{.RestartCount}}'` |
| Container A không gọi được B | `docker exec A ping B`, `docker network inspect` |
| Web trả 502/503 | `docker ps -a` (kiểm tra backend còn `Up` không), `docker logs` backend |
| Ghi file bị `Permission denied` | `docker exec <container> id`, `ls -lan` trên host |
| Đĩa báo `no space left` | `docker system df`, `df -h /var/lib/docker` |
| Container chậm bất thường | `docker stats`, sau đó đọc cgroup trực tiếp |
| Container bị kill đột ngột (exit 137) | `dmesg -T \| grep -i killed`, `docker inspect --format '{{.State.OOMKilled}}'` |
