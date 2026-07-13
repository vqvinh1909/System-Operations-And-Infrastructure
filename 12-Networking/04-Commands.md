---
title: "04 - Commands"
module: 12
tags: [docker, sysops-infra, module-12, commands, reference]
---

# 04 - Commands: Network Driver, Port Mapping, DNS

## 1. Quản lý network

```bash
# Liet ke toan bo network hien co
docker network ls

# Xem chi tiet: subnet, gateway, container dang gan, driver
docker network inspect mynet

# Tao user-defined bridge network (best practice cho moi app multi-container)
docker network create mynet

# Tao voi subnet/gateway tuy chinh
docker network create --subnet=172.28.0.0/16 --gateway=172.28.0.1 mynet

# Tao network driver macvlan gan voi interface vat ly eth0
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 macvlan-net

# Xoa network (chi xoa duoc khi khong con container nao dang dung)
docker network rm mynet

# Don cac network khong con container nao dung
docker network prune
```

## 2. Gắn/gỡ container khỏi network

```bash
# Chay container va gan luon vao user-defined network
docker run -d --name web --network mynet nginx:1.27-alpine

# Gan them mot network khac cho container DANG CHAY (co the thuoc nhieu network)
docker network connect another-net web

# Dat network alias - container co the duoc goi bang nhieu ten trong cung network
docker network connect --alias webapp mynet web

# Go container khoi mot network
docker network disconnect mynet web
```

## 3. Port mapping: publish vs expose

```bash
# Publish port ra host - client ben ngoai goi duoc
docker run -d --name web -p 8080:80 nginx:1.27-alpine

# Publish tren dia chi cu the (khong bind 0.0.0.0)
docker run -d --name web -p 127.0.0.1:8080:80 nginx:1.27-alpine

# Publish nhieu port cung luc
docker run -d --name web -p 8080:80 -p 8443:443 nginx:1.27-alpine

# Publish UDP thay vi TCP mac dinh
docker run -d --name dns-app -p 53:53/udp mydns:latest

# Publish tat ca port da EXPOSE trong Dockerfile ra port ngau nhien tren host
docker run -d --name web -P nginx:1.27-alpine

# Xem port nao dang duoc map, ra sao
docker port web
```

## 4. Kiểm chứng ở tầng Linux (đúng kỹ năng Sysadmin đã học)

```bash
# Xem bridge Docker da tao
ip link show type bridge
brctl show          # neu co cai bridge-utils

# Xem veth pair cua mot container cu the
docker inspect web --format '{{.NetworkSettings.SandboxKey}}'
sudo nsenter --net=<sandbox-key-path> ip addr

# Xem rule DNAT that su duoc Docker chen vao iptables
sudo iptables -t nat -L DOCKER -n --line-numbers

# Xac nhan ip_forward da bat (dieu kien can de traffic di qua bridge)
sysctl net.ipv4.ip_forward

# Xem resolv.conf ben trong container - xac nhan dung 127.0.0.11
docker exec web cat /etc/resolv.conf
```

## 5. Debug DNS/kết nối giữa các container

```bash
# Test container A goi duoc container B bang ten khong (cung user-defined network)
docker exec containerA ping -c 2 containerB
docker exec containerA getent hosts containerB

# Test tu ben trong container ra Internet
docker exec containerA curl -sSf https://example.com -o /dev/null -w '%{http_code}\n'

# Xem IP noi bo cua tung container trong network
docker network inspect mynet --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

## 6. `--network host` và `--network none`

```bash
# Host network - khong NAT, khong -p, dung chung netns voi host
docker run -d --name monitor --network host netdata/netdata

# None network - co lap hoan toan, chi co loopback
docker run -d --name batch-job --network none myjob:latest
```

Tiếp theo: [[05-Labs]] để thực hành trực tiếp các lệnh này.
