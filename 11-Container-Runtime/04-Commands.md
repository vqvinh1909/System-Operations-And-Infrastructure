---
title: "04 - Commands"
module: 11
tags: [docker, sysops-infra, module-11, commands, reference]
---

# 04 - Commands: Vòng đời, Resource Limit, Logs, Exec, Inspect, Monitoring

## 1. Vòng đời container

```bash
docker create --name web nginx:1.27-alpine
docker start web
docker run -d --name web -p 8080:80 nginx:1.27-alpine   # create + start gop lai

docker stop web              # SIGTERM, grace period 10s mac dinh, roi SIGKILL
docker stop -t 30 web        # grace period tuy chinh 30s
docker kill web               # SIGKILL ngay lap tuc
docker kill --signal=SIGHUP web   # gui signal khac de reload config

docker pause web
docker unpause web

docker restart web           # stop + start
docker restart -t 20 web

docker rm web                 # xoa container da dung
docker rm -f web               # force: kill roi xoa
docker container prune        # xoa tat ca container da exited
```

## 2. Resource Limit — CPU/Memory

```bash
# Gioi han cung 512MB RAM, khong dung swap them
docker run -d --name app --memory=512m --memory-swap=512m myapp:latest

# Nguong mem 256MB - chi ap dung khi host chiu ap luc RAM
docker run -d --name app --memory=512m --memory-reservation=256m myapp:latest

# Gioi han 1.5 CPU core (quota/period trong cgroups v2)
docker run -d --name app --cpus=1.5 myapp:latest

# Trong so CPU tuong doi khi tranh chap (mac dinh 1024)
docker run -d --name app --cpu-shares=512 myapp:latest

# Ghim container chi chay tren core 0 va 1
docker run -d --name app --cpuset-cpus="0,1" myapp:latest

# Kiem tra gia tri thuc te da ap dung vao cgroup
docker inspect app --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'

# Xac nhan container co bi OOM kill khong
docker inspect app --format '{{.State.OOMKilled}} {{.State.ExitCode}}'
```

## 3. Restart Policy

```bash
docker run -d --name worker --restart=on-failure:5 worker:latest
docker run -d --name proxy --restart=always reverse-proxy:latest
docker run -d --name api --restart=unless-stopped api:latest

# Xem policy hien tai cua mot container da chay
docker inspect api --format '{{.HostConfig.RestartPolicy}}'

# Doi restart policy cho container da ton tai (khong can tao lai)
docker update --restart=unless-stopped api
```

## 4. Logging — giới hạn dung lượng

```bash
# Per-container: gioi han log ngay khi tao container
docker run -d --name app \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp:latest

# Xem log
docker logs app
docker logs -f app                # theo doi realtime
docker logs --tail 100 app
docker logs --since 10m app       # log trong 10 phut gan nhat

# Kiem tra file log that su nam o dau va kich thuoc hien tai
docker inspect app --format '{{.LogPath}}'
```

Cấu hình toàn cục cho mọi container mới trong `/etc/docker/daemon.json`:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

## 5. Debug: `docker exec` vs `docker attach`

```bash
# An toan - tao tien trinh moi, khong dung cham tien trinh chinh
docker exec -it app /bin/sh
docker exec app env
docker exec app cat /etc/resolv.conf

# Rui ro hon - ket noi truc tiep vao stdin/stdout PID 1
docker attach app
# Thoat dung cach (KHONG dung Ctrl+C): Ctrl+P roi Ctrl+Q de detach an toan
```

## 6. `docker inspect` với Go template

```bash
docker inspect app --format '{{.State.Status}}'
docker inspect app --format '{{.State.Pid}}'
docker inspect app --format '{{.NetworkSettings.IPAddress}}'
docker inspect app --format '{{json .State}}' | python3 -m json.tool
docker inspect app --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
```

## 7. `--init` để tránh vấn đề signal/zombie

```bash
docker run -d --name app --init myapp:latest
docker inspect app --format '{{.Path}}'   # xac nhan tini duoc chen vao lam PID 1
```

## 8. Giám sát vận hành: `docker events`, `docker stats`

```bash
# Theo doi realtime moi su kien tren daemon
docker events

# Loc theo loai su kien va container
docker events --filter 'event=die'
docker events --filter 'container=app'
docker events --since '2026-07-13T00:00:00'

# Xem tai nguyen realtime (giong top nhung cho container)
docker stats

# Chi xem 1 container, khong lap lai lien tuc (mot lan snapshot)
docker stats --no-stream app

# Dinh dang output gon de dua vao script
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

## 9. Dọn dẹp: `docker system df`, `docker system prune`

```bash
# Xem tong dung luong dang chiem: images, containers, volumes, build cache
docker system df
docker system df -v      # chi tiet tung doi tuong

# Don container da exited, network khong dung, dangling image, build cache
docker system prune

# Mo rong: don ca image khong con container nao dung (khong chi dangling)
docker system prune -a

# CANH BAO: them --volumes se xoa ca volume khong duoc container nao tham chieu
docker system prune -a --volumes

# Don rieng tung loai, kiem soat tot hon toan bo
docker container prune
docker image prune -a
docker volume prune
docker network prune
docker builder prune
```

**Nguyên tắc an toàn khi dùng `prune` trong production**: luôn chạy `docker system df -v` trước để biết chính xác cái gì sắp bị xóa, không bao giờ dùng `--volumes` trên server production mà không kiểm tra kỹ danh sách volume trước — volume có thể chứa dữ liệu quan trọng dù hiện tại không container nào đang tham chiếu tới nó (ví dụ volume backup tạm thời tách rời container).

Tiếp theo: [[05-Labs]] để thực hành trực tiếp các lệnh này.
