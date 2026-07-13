---
title: "04 - Commands"
module: 13
tags: [docker, sysops-infra, module-13, commands, reference]
---

# 04 - Commands: Volume, Bind Mount, tmpfs, Backup/Restore

## 1. Quản lý named volume

```bash
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker volume rm mydata

# Xoa tat ca volume khong con container nao tham chieu
docker volume prune

# Tao volume voi driver/option tuy chinh (vi du gioi han hoac dung driver khac local)
docker volume create --driver local \
  --opt type=none --opt device=/data/mydata --opt o=bind mydata
```

## 2. Cú pháp `-v` và `--mount`

```bash
# Cu phap ngan gon "-v" (source:destination[:mode])
docker run -v mydata:/var/lib/mysql mysql:8.4
docker run -v /home/vinh/code:/app myapp:latest
docker run -v mydata:/var/lib/mysql:ro myapp:latest   # chi doc

# Cu phap day du "--mount" (ro rang tung phan, khuyen nghi cho script/automation)
docker run --mount type=volume,source=mydata,target=/var/lib/mysql mysql:8.4
docker run --mount type=bind,source=/home/vinh/code,target=/app myapp:latest
docker run --mount type=bind,source=/home/vinh/code,target=/app,readonly myapp:latest
```

**Khác biệt quan trọng cần nhớ**: với `-v`, nếu tên "source" không tồn tại như một volume và không phải đường dẫn tuyệt đối bắt đầu bằng `/`, Docker **tự tạo named volume mới** với đúng tên đó. Với `--mount`, cú pháp buộc bạn phải khai báo tường minh `type=volume` hay `type=bind` — giảm rủi ro gõ nhầm dẫn tới hành vi không mong muốn (đây là lý do nhiều tổ chức khuyến nghị `--mount` cho script tự động hóa, dù `-v` gọn hơn khi gõ tay).

## 3. tmpfs

```bash
docker run --tmpfs /app/cache myapp:latest
docker run --tmpfs /app/cache:size=100m,mode=1777 myapp:latest
docker run --mount type=tmpfs,destination=/app/cache,tmpfs-size=104857600 myapp:latest
```

## 4. Kiểm tra mount thực tế của một container

```bash
docker inspect <container> --format '{{json .Mounts}}' | python3 -m json.tool

# Xem duong dan vat ly that cua named volume tren host
docker volume inspect mydata --format '{{.Mountpoint}}'
sudo ls -la $(docker volume inspect mydata --format '{{.Mountpoint}}')
```

## 5. Backup volume ra file tar.gz

```bash
mkdir -p ./backup

docker run --rm \
  -v mydata:/source:ro \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/mydata-$(date +%Y%m%d-%H%M).tar.gz -C /source .

ls -lh ./backup
```

## 6. Restore volume từ file tar.gz

```bash
docker volume create mydata-restored

docker run --rm \
  -v mydata-restored:/target \
  -v $(pwd)/backup:/backup \
  alpine tar xzf /backup/mydata-20260713-0900.tar.gz -C /target

# Xac nhan du lieu da khoi phuc dung
docker run --rm -v mydata-restored:/target alpine ls -la /target
```

## 7. Sao chép dữ liệu giữa 2 volume (không qua tar, dùng trực tiếp `cp`)

```bash
docker run --rm \
  -v mydata:/from:ro \
  -v mydata-copy:/to \
  alpine sh -c "cp -a /from/. /to/"
```

## 8. Kiểm tra và xử lý anonymous volume

```bash
# Liet ke volume, cot NAME la hash ngau nhien -> dau hieu anonymous volume
docker volume ls

# Xem container nao dang tham chieu mot volume cu the (kiem tra truoc khi xoa)
docker ps -a --filter volume=mydata

# Xoa container KEM anonymous volume gan voi no (can trong, khong the hoan tac)
docker rm -v <container>

# Don anonymous volume mo coi (khong con container nao tham chieu)
docker volume prune
```

Tiếp theo: [[05-Labs]] để thực hành trực tiếp các lệnh này, đặc biệt là quy trình backup/restore.
