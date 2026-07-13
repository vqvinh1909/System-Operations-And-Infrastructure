---
title: "06 - Troubleshooting (Nội dung chính)"
module: 19
tags: [docker, sysops-infra, module-19, troubleshooting, debugging, performance]
---

# 06 — Troubleshooting Chuyên Sâu: Container, Network, Storage, Performance

> [!info] Đây là nội dung chính của Module 19
> File này dài và chi tiết hơn hẳn các file `06-Troubleshooting.md` ở những module trước — đúng như vai trò của Module 19 trong khóa học: tổng hợp và đào sâu toàn bộ kỹ năng debug. Mỗi kịch bản dưới đây trình bày theo cùng cấu trúc: **Triệu chứng** (những gì bạn quan sát được đầu tiên) → **Chẩn đoán** (lệnh cụ thể để thu thập bằng chứng) → **Nguyên nhân gốc** → **Khắc phục** → **Bài học rút ra** (áp dụng được cho tình huống tương tự khác).

---

## Phần A — Container Debugging

### A1. Container thoát ngay lập tức với exit code 1, log rỗng

**Triệu chứng:** `docker run` xong, `docker ps -a` cho thấy container ở trạng thái `Exited (1)` chỉ sau vài trăm mili-giây. `docker logs` không in ra gì, hoặc chỉ một dòng rất ngắn không đủ thông tin.

**Chẩn đoán:**
```bash
docker inspect --format '{{.State.ExitCode}}' <container>
docker inspect --format '{{.Config.Entrypoint}} {{.Config.Cmd}}' <container>
docker run --rm -it --entrypoint sh <image>   # ghi de entrypoint, vao shell thay vi chay app
```

**Nguyên nhân gốc:** thường là một trong ba khả năng — (1) entrypoint/binary được khai báo trong Dockerfile không tồn tại đúng đường dẫn trong image thực tế (lỗi hay gặp khi build multi-stage, quên copy binary sang stage cuối); (2) thiếu biến môi trường bắt buộc khiến ứng dụng tự thoát ngay khi validate config lúc khởi động; (3) file cấu hình ứng dụng cần có nhưng chưa được mount/copy vào đúng vị trí.

**Khắc phục:** dùng `--entrypoint sh` (hoặc `bash`) để vào được bên trong image mà không chạy tiến trình chính, từ đó tự tay chạy thử lệnh gốc để thấy lỗi thật (log rỗng thường là vì tiến trình chết quá nhanh, trước khi logging driver kịp buffer đủ dữ liệu để hiển thị).

**Bài học rút ra:** log rỗng không có nghĩa "không có lỗi" — nó thường có nghĩa "lỗi xảy ra trước khi có gì để log". Luôn có phương án B (ghi đè entrypoint) khi log không đủ thông tin.

### A2. Container restart loop, mỗi lần sống được khoảng 10-15 giây

**Triệu chứng:** `docker ps` liên tục thấy container ở trạng thái `Up X seconds` rồi lại `Restarting`, lặp lại vô hạn.

**Chẩn đoán:**
```bash
docker events --since 10m --filter container=<container>
docker inspect --format '{{.RestartCount}}' <container>
docker logs --tail 200 <container>   # log cua LAN CHAY GAN NHAT truoc khi restart
```

**Nguyên nhân gốc:** khoảng thời gian sống nhất quán (luôn ~10-15 giây) là manh mối quan trọng — thường là ứng dụng khởi động thành công, chạy được một đoạn ngắn, rồi crash tại một bước cụ thể lặp lại mỗi lần (ví dụ kết nối database timeout sau 10 giây do database chưa sẵn sàng, hoặc một tác vụ nền chạy theo lịch mỗi 10-15 giây bị lỗi).

**Khắc phục:** đọc kỹ log của **đúng lần chạy gần nhất** (không phải log tích lũy từ đầu, dễ gây nhiễu) để xác định chính xác bước nào crash. Nếu do phụ thuộc dịch vụ khác chưa sẵn sàng (ví dụ database), cân nhắc thêm cơ chế chờ (`depends_on` với `condition: service_healthy` trong Docker Compose, hoặc script wait-for tự viết) thay vì để ứng dụng tự crash và dựa vào restart policy để "thử lại".

**Bài học rút ra:** thời gian sống nhất quán trước khi crash gần như luôn chỉ ra một bước cụ thể trong logic khởi động, không phải lỗi ngẫu nhiên — tận dụng tính nhất quán đó để khoanh vùng nhanh.

### A3. Container ở trạng thái `Up` và `healthy`, nhưng ứng dụng bên trong vẫn không phản hồi

**Triệu chứng:** mọi chỉ số bề ngoài đều "xanh" — `docker ps` báo `Up`, healthcheck báo `healthy` — nhưng request thật tới ứng dụng vẫn treo hoặc timeout.

**Chẩn đoán:**
```bash
docker exec -it <container> sh
# ben trong container:
ps aux                     # tien trinh chinh co thuc su con song khong, co bi "zombie" khong
ss -tlnp                   # co dang lang nghe dung port khong
curl -v http://localhost:<port>/health   # goi truc tiep TU BEN TRONG, loai tru yeu to network ben ngoai
```

**Nguyên nhân gốc:** đây là trường hợp khó nhất trong tầng Container vì mọi cơ chế giám sát tự động (healthcheck) đều dựa trên logic do chính bạn định nghĩa — nếu healthcheck chỉ kiểm tra "port có mở không" mà không kiểm tra "ứng dụng có xử lý được request thật không", nó sẽ báo `healthy` sai. Nguyên nhân thường gặp: thread pool của ứng dụng bị cạn (deadlock hoặc connection pool tới database bị full), tiến trình chính vẫn sống nhưng không còn xử lý được request mới.

**Khắc phục:** viết lại healthcheck để gọi một endpoint thật sự phản ánh khả năng xử lý request (không chỉ kiểm tra port mở), ví dụ endpoint tự thực hiện một truy vấn nhẹ tới database rồi mới trả `200 OK`.

**Bài học rút ra:** healthcheck chỉ tốt bằng logic bạn viết cho nó — một healthcheck hời hợt còn nguy hiểm hơn không có healthcheck, vì nó tạo cảm giác an toàn giả.

### A4. `docker exec` vào container báo lỗi, dù `docker ps` vẫn thấy container `Up`

**Triệu chứng:** `docker exec -it <container> sh` trả về lỗi kiểu `OCI runtime exec failed` hoặc treo vô thời hạn không vào được shell.

**Chẩn đoán:**
```bash
docker inspect --format '{{.State.Status}} {{.State.Pid}}' <container>
sudo ls /proc/<pid>   # kiem tra tien trinh chinh co thuc su ton tai o muc kernel khong
```

**Nguyên nhân gốc:** thường gặp khi container đang trong trạng thái chuyển tiếp (đang bị `docker stop` xử lý dở, hoặc container runtime bên dưới — containerd/runc — đang gặp vấn đề riêng dù Docker daemon vẫn báo trạng thái cũ chưa cập nhật kịp), hoặc image không có shell nào cài sẵn (phổ biến với image tối giản kiểu distroless, `scratch` — vốn là lựa chọn tốt cho bảo mật nhưng đánh đổi mất khả năng exec vào debug trực tiếp).

**Khắc phục:** nếu do trạng thái chuyển tiếp, đợi và kiểm tra lại `docker ps`. Nếu do image không có shell, cân nhắc dùng `docker debug` (nếu có sẵn) hoặc tạo một image debug riêng có cùng base nhưng thêm shell, dùng tạm trong lúc điều tra — không nên thêm shell vĩnh viễn vào image production chỉ để tiện debug, vì điều đó đi ngược nguyên tắc tối giản bề mặt tấn công đã học ở Module 18.

**Bài học rút ra:** đánh đổi giữa "image tối giản, an toàn hơn" và "image dễ debug hơn" là một quyết định kiến trúc có chủ đích, không phải sự cố — cần có kế hoạch debug thay thế (ví dụ log tập trung đầy đủ hơn từ Module 17) khi chọn image tối giản.

---

## Phần B — Network Debugging

### B1. Hai container cùng `docker-compose.yml` không gọi được nhau

**Triệu chứng:** ứng dụng web báo lỗi kết nối tới database dù cả hai container đều `Up`, cùng khởi động từ một file Compose.

**Chẩn đoán:** áp dụng đúng 4 bước ở [[03-Architecture]] Sơ đồ 2. Điểm kiểm tra riêng cho Docker Compose:
```bash
docker compose ps
docker network inspect <ten-project>_default    # Compose tu tao network rieng theo ten thu muc project
```

**Nguyên nhân gốc phổ biến nhất:** dùng sai **tên service** làm hostname trong connection string của ứng dụng (ví dụ ứng dụng cấu hình trỏ tới `localhost:5432` thay vì `db:5432` — trong đó `db` là tên service database khai báo trong `docker-compose.yml`). `localhost` bên trong container luôn trỏ về chính container đó, không bao giờ trỏ sang container khác dù chạy chung Compose.

**Khắc phục:** sửa connection string dùng đúng tên service làm hostname — Docker Compose tự tạo DNS nội bộ ánh xạ tên service sang IP container trong cùng network mặc định.

**Bài học rút ra:** đây là lỗi phổ biến nhất trong toàn bộ phần Network Debugging, gặp lại xuyên suốt nhiều module trước (Module 17 khi cấu hình datasource Grafana). Luôn kiểm tra hostname trong cấu hình kết nối trước khi đào sâu vào iptables/bridge.

### B2. Container gọi được ra internet, nhưng không gọi được container khác cùng host

**Triệu chứng:** từ bên trong container, `curl https://google.com` thành công, nhưng `curl http://<container-khac>` thất bại.

**Chẩn đoán:**
```bash
docker network inspect <network> --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
docker exec <containerA> cat /etc/resolv.conf   # xac nhan dang dung DNS noi bo cua Docker (127.0.0.11)
```

**Nguyên nhân gốc:** hai container không nằm cùng một Docker network (mỗi network là một bridge cô lập riêng) — việc gọi ra internet thành công không liên quan gì tới việc hai container có "thấy" nhau hay không, vì internet đi qua NAT ra ngoài, còn giao tiếp nội bộ container-tới-container phụ thuộc hoàn toàn vào việc chúng có chung network hay không.

**Khắc phục:** thêm cả hai container vào cùng một network tường minh (`docker network connect <network> <container>`, hoặc khai báo đúng trong Compose), không dựa vào network `bridge` mặc định (mạng mặc định của Docker không có DNS tự động resolve theo tên container như network tự tạo).

**Bài học rút ra:** "gọi được ra ngoài" và "gọi được nội bộ" là hai bài toán network hoàn toàn tách biệt trong Docker — đừng suy luận cái này từ cái kia.

### B3. Port publish đúng cấu hình nhưng không truy cập được từ máy khác trong mạng LAN

**Triệu chứng:** `curl http://localhost:8080` từ chính host chạy Docker thành công, nhưng từ máy khác trong cùng mạng LAN gọi tới `http://<ip-host>:8080` bị timeout.

**Chẩn đoán:**
```bash
sudo ss -tlnp | grep 8080                 # xac nhan dang lang nghe tren 0.0.0.0, khong chi 127.0.0.1
sudo iptables -L -n | grep 8080           # kiem tra firewall host co chan khong
sudo ufw status 2>/dev/null               # neu dung ufw
```

**Nguyên nhân gốc:** thường không phải lỗi Docker mà là **firewall của chính host** (ufw, firewalld, security group nếu chạy cloud) chặn port đó ở tầng ngoài Docker hoàn toàn quản lý — Docker tự thêm rule iptables riêng để NAT traffic vào container, nhưng nếu có một tầng firewall khác (ví dụ ufw ở chế độ mặc định deny incoming) đứng trước, nó vẫn có thể chặn traffic trước khi tới được rule của Docker.

**Khắc phục:** mở đúng port trên firewall của host (`sudo ufw allow 8080/tcp` hoặc tương đương), đồng thời nếu chạy trên cloud, kiểm tra thêm security group/network ACL ở tầng hạ tầng cloud — đây là một tầng firewall thứ ba, hoàn toàn bên ngoài cả host lẫn Docker.

**Bài học rút ra:** khi port hoạt động từ `localhost` nhưng không từ bên ngoài, luôn nghi ngờ theo thứ tự: firewall host → firewall/security group hạ tầng → cuối cùng mới nghi ngờ tới cấu hình Docker, vì Docker publish port đúng gần như luôn hoạt động nhất quán ở cả `localhost` lẫn từ ngoài nếu không có gì chặn thêm.

### B4. DNS bên trong container resolve được tên container nội bộ, nhưng không resolve được domain internet

**Triệu chứng:** `docker exec <container> ping <container-khac>` thành công, nhưng `docker exec <container> ping google.com` báo lỗi `Name or service not known`.

**Chẩn đoán:**
```bash
docker exec <container> cat /etc/resolv.conf
docker exec <container> nslookup google.com
cat /etc/docker/daemon.json | grep -i dns
```

**Nguyên nhân gốc:** DNS nội bộ Docker (`127.0.0.11`) chịu trách nhiệm resolve tên container, đồng thời forward các truy vấn không phải tên container ra DNS server thật — nếu host đang ở mạng có DNS server nội bộ đặc thù (ví dụ VPN công ty, DNS server nội bộ chỉ host mới truy cập được) mà Docker daemon không kế thừa đúng cấu hình DNS đó, container sẽ không resolve được domain bên ngoài dù bản thân daemon/host vẫn resolve bình thường.

**Khắc phục:** khai báo tường minh DNS server trong `daemon.json` (`"dns": ["8.8.8.8", "1.1.1.1"]` hoặc DNS nội bộ công ty nếu bắt buộc), restart Docker daemon để áp dụng.

**Bài học rút ra:** DNS nội bộ container (resolve tên container) và DNS ra ngoài internet là hai cơ chế xếp chồng lên nhau trong cùng `/etc/resolv.conf` của container — lỗi ở cái này không đồng nghĩa lỗi ở cái kia, cần test riêng từng loại truy vấn.

---

## Phần C — Storage Debugging

### C1. Ứng dụng ghi log/file báo `Permission denied` dù bind mount đúng đường dẫn

**Triệu chứng:** container khởi động, nhưng ứng dụng bên trong báo lỗi không ghi được vào thư mục đã `-v /data/app:/app/data`.

**Chẩn đoán:**
```bash
docker exec <container> id                      # UID/GID tien trinh dang chay ben trong
ls -lan /data/app                                # UID/GID that su so huu thu muc tren HOST
```

**Nguyên nhân gốc:** kể từ khi thực hành bảo mật khuyến nghị chạy ứng dụng bằng non-root user bên trong container (đã học ở Module 18), UID bên trong container (ví dụ UID 1000) thường **không trùng** với UID sở hữu thư mục trên host (thường là UID của user tạo thư mục đó, hoặc `root`). Bind mount không tự động "dịch" quyền — nó giữ nguyên UID số nguyên, và Linux kiểm tra quyền theo UID số nguyên đó, không quan tâm UID đó "là ai" ở mỗi phía.

**Khắc phục:** đồng bộ UID hai phía — hoặc `chown` thư mục host cho đúng UID mà container dùng (`sudo chown -R 1000:1000 /data/app`), hoặc chỉ định rõ UID khi build/run container để khớp với UID đã có sẵn trên host, tùy quy trình vận hành nào phù hợp hơn với tổ chức.

**Bài học rút ra:** đây chính là "chi phí" thường bị bỏ qua khi áp dụng khuyến nghị bảo mật "chạy non-root" — cần một bước đồng bộ UID rõ ràng trong quy trình deploy, không phải lỗi ngẫu nhiên.

### C2. `docker system df` cho thấy dung lượng lớn nằm ở "build cache" không ai để ý

**Triệu chứng:** đĩa host báo gần đầy dù `docker ps -a` không thấy nhiều container, `docker images` cũng không thấy nhiều image.

**Chẩn đoán:**
```bash
docker system df -v
docker builder du     # rieng cho build cache cua BuildKit
```

**Nguyên nhân gốc:** BuildKit (engine build mặc định của Docker hiện nay) giữ lại cache của từng layer đã build qua nhiều lần `docker build` để tăng tốc build sau — cache này tích lũy âm thầm theo thời gian, đặc biệt trên máy CI/CD build liên tục nhiều lần mỗi ngày, và **không hiển thị** trong `docker images` thông thường vì nó không phải image hoàn chỉnh.

**Khắc phục:**
```bash
docker builder prune -a           # don rieng build cache
docker system prune -a --volumes  # don tong the, CAN THAN voi volume dang co du lieu quan trong
```

**Bài học rút ra:** "dung lượng Docker chiếm dụng" không chỉ là image + container + volume như trực giác thông thường — build cache là một nguồn chiếm dụng độc lập, cần dọn định kỳ riêng, đặc biệt trên máy CI/CD.

### C3. Container mới không khởi động được sau khi host vừa hết dung lượng đĩa

**Triệu chứng:** sau khi giải phóng đĩa (ví dụ đã xóa bớt file lớn không liên quan), `docker run` container mới vẫn báo lỗi liên quan tới storage, dù `df -h` giờ đã còn nhiều dung lượng trống.

**Chẩn đoán:**
```bash
docker info | grep -i "storage driver"
sudo journalctl -u docker --since "1 hour ago" | grep -i overlay
```

**Nguyên nhân gốc:** khi đĩa hết dung lượng đột ngột trong lúc Docker đang ghi dữ liệu (ví dụ giữa lúc pull một layer image, hoặc container đang ghi vào một layer overlay2), thao tác ghi có thể bị gián đoạn nửa chừng, để lại dữ liệu overlay2 ở trạng thái không toàn vẹn — vấn đề này **không tự khỏi** chỉ vì đĩa đã có chỗ trống trở lại, vì phần dữ liệu hỏng vẫn còn nằm đó.

**Khắc phục:** kiểm tra log daemon tìm dấu hiệu lỗi overlay2 cụ thể, thường cần xóa thủ công đúng layer/container bị ảnh hưởng (`docker rm -f` container liên quan, `docker system prune` để dọn layer mồ côi), trường hợp nghiêm trọng hơn cần restart Docker daemon để nó tự kiểm tra lại trạng thái, hiếm khi cần tới việc xóa toàn bộ `/var/lib/docker` và pull lại từ đầu (biện pháp cuối cùng).

**Bài học rút ra:** hết dung lượng đĩa giữa lúc Docker đang ghi dữ liệu là một trong số ít tình huống mà "nguyên nhân đã hết" không đồng nghĩa "hậu quả đã hết" — luôn kiểm tra log daemon sau một sự cố đầy đĩa, không chỉ tin rằng dọn đĩa xong là xử lý xong.

### C4. Volume vẫn còn dữ liệu cũ dù đã đổi image sang version mới, tưởng nhầm là "bug của app"

**Triệu chứng:** deploy version mới của ứng dụng, nhưng dữ liệu hiển thị vẫn là dữ liệu cũ như trước khi deploy — nghi ngờ ban đầu thường đổ cho lỗi code.

**Chẩn đoán:**
```bash
docker inspect --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' <container>
docker volume inspect <volume>
```

**Nguyên nhân gốc:** named volume trong Docker **có vòng đời độc lập với container** — khi bạn `docker compose up` với image mới, container cũ bị xóa và tạo lại, nhưng volume dữ liệu (được khai báo `volumes:` trong Compose, không phải phần image) vẫn được tái sử dụng y nguyên, đúng như thiết kế (đây là tính năng, không phải lỗi — dữ liệu database không nên mất mỗi lần deploy). Nhầm lẫn xảy ra khi người vận hành quên rằng "đổi image" không đồng nghĩa "đổi dữ liệu", đặc biệt khi debug và nghi ngờ sai chỗ.

**Khắc phục:** không phải lúc nào cũng cần khắc phục — cần **nhận diện đúng** đây là hành vi thiết kế đúng của volume trước khi tốn thời gian tìm "bug" không tồn tại trong code. Nếu thật sự cần reset dữ liệu (ví dụ môi trường test), xóa tường minh volume đó bằng `docker compose down -v` hoặc `docker volume rm`.

**Bài học rút ra:** không phải mọi "sự cố" đều là lỗi — một phần quan trọng của troubleshooting là phân biệt được hành vi thiết kế đúng nhưng gây bất ngờ, với lỗi thật sự cần sửa.

---

## Phần D — Performance Analysis

### D1. Container bị kill đột ngột, exit code 137

**Triệu chứng:** container tự nhiên chuyển sang `Exited (137)` không có log lỗi ứng dụng rõ ràng ngay trước đó.

**Chẩn đoán:**
```bash
docker inspect --format '{{.State.OOMKilled}}' <container>
dmesg -T | grep -i "killed process"
```

**Nguyên nhân gốc:** exit code 137 = 128 + 9 (`SIGKILL`) — gần như luôn là dấu hiệu của **OOM Killer** (cơ chế kernel tự động giết tiến trình khi hệ thống/cgroup cạn kiệt RAM). Hai kịch bản con cần phân biệt: (a) container có đặt `--memory` limit và ứng dụng vượt đúng limit riêng của nó — `OOMKilled: true` trong inspect; (b) host tổng thể cạn RAM (không phải riêng container này vượt limit của nó) — kernel OOM Killer chọn tiến trình "đáng bị giết nhất" theo thuật toán riêng, có thể là container hoàn toàn không liên quan tới nguyên nhân gây thiếu RAM.

**Khắc phục:** nếu (a), tăng `--memory` limit hoặc tối ưu ứng dụng giảm tiêu thụ RAM (memory leak là nghi vấn hàng đầu nếu RAM tăng dần đều theo thời gian trước khi bị kill). Nếu (b), cần nhìn tổng thể toàn bộ container trên host, xem lại kế hoạch capacity (có đang chạy quá nhiều workload so với RAM vật lý thật sự có).

**Bài học rút ra:** exit code 137 luôn cần đối chiếu với `dmesg` để xác nhận đúng là OOM (không phải một signal khác trùng hợp), và cần phân biệt rõ hai kịch bản con ở trên vì cách khắc phục hoàn toàn khác nhau.

### D2. CPU limit đặt đúng nhưng ứng dụng vẫn "cảm giác" chậm hơn dự kiến

**Triệu chứng:** đặt `--cpus=1` cho một ứng dụng, `docker stats` cho thấy CPU dùng gần 100%, nhưng độ trễ xử lý request cao hơn hẳn so với khi test không giới hạn CPU.

**Chẩn đoán:**
```bash
CID=$(docker inspect --format '{{.Id}}' <container>)
cat /sys/fs/cgroup/system.slice/docker-${CID}.scope/cpu.stat
# chu y truong nr_throttled va throttled_usec
```

**Nguyên nhân gốc:** `cpu.stat` cho thấy số lần container bị **throttle** (tạm dừng thực thi để không vượt quota CPU trong mỗi chu kỳ) và tổng thời gian bị throttle. Nếu `nr_throttled` cao, ứng dụng đang bị "ngắt quãng" liên tục dù trung bình CPU usage nhìn qua `docker stats` có vẻ chưa chạm 100% — vì cách CPU quota hoạt động theo từng chu kỳ ngắn (thường 100ms), một ứng dụng đa luồng có thể dùng hết quota rất nhanh trong đầu chu kỳ rồi bị đóng băng phần còn lại, tạo ra độ trễ giật cục dù số liệu trung bình trông "bình thường".

**Khắc phục:** với ứng dụng đa luồng nhạy cảm độ trễ, cân nhắc tăng `--cpus` hoặc chuyển sang dùng `--cpuset-cpus` (gán cứng vào số lõi CPU cụ thể thay vì chia sẻ theo quota theo thời gian) nếu phù hợp với kiến trúc hạ tầng.

**Bài học rút ra:** `docker stats` báo % CPU trung bình có thể che giấu hiện tượng throttle giật cục — khi nghi ngờ vấn đề độ trễ liên quan CPU, luôn đọc thêm `cpu.stat` thay vì chỉ tin số % trung bình.

### D3. Nhiều container đều báo hiệu năng kém cùng lúc, không container nào chạm limit riêng

**Triệu chứng:** `docker stats` của từng container riêng lẻ đều cho thấy số liệu "bình thường" (không container nào gần limit CPU/RAM riêng), nhưng toàn bộ ứng dụng trên host đều chậm.

**Chẩn đoán:**
```bash
vmstat 1 5
top
iostat -x 1 5
```

**Nguyên nhân gốc:** đây là trường hợp kinh điển của **host quá tải tổng thể** dù từng container "trông" ổn — tổng nhu cầu CPU/RAM/IO của tất cả container cộng lại vượt quá tài nguyên vật lý thật sự có, gây tranh chấp tài nguyên (resource contention) ở tầng scheduler của kernel mà số liệu riêng lẻ từng container không thể hiện rõ. `vmstat` cho thấy dấu hiệu qua cột `r` (số tiến trình chờ CPU) cao bất thường, hoặc `iostat` cho thấy `%util` của thiết bị đĩa gần 100% dù không container riêng lẻ nào ghi log nhiều.

**Khắc phục:** đây không phải sự cố có thể sửa bằng cách điều chỉnh cấu hình một container cụ thể — cần xem lại capacity planning tổng thể: giảm số lượng container đồng thời trên host, tăng tài nguyên vật lý (scale up), hoặc phân tán container sang nhiều host hơn (scale out).

**Bài học rút ra:** đây chính là lý do Bước 3 ("đối chiếu tài nguyên host") ở [[03-Architecture]] Sơ đồ 3 không phải bước tùy chọn — nếu bỏ qua bước này, bạn có thể kiểm tra từng container mãi mà không bao giờ tìm ra nguyên nhân thật, vì vấn đề nằm ở tổng thể chứ không ở bất kỳ container riêng lẻ nào.

### D4. Ứng dụng chậm chỉ vào một khung giờ cố định mỗi ngày

**Triệu chứng:** hiệu năng bình thường suốt cả ngày, nhưng luôn chậm đột ngột vào một khung giờ lặp lại (ví dụ 2h sáng mỗi ngày).

**Chẩn đoán:**
```bash
docker events --since 24h | grep -i "02:0"     # tim su kien Docker trong dung khung gio nghi van
crontab -l                                       # kiem tra co cron job nao chay dung gio do
docker ps -a --filter "status=exited"           # co container batch job nao chay xong roi tu thoat
```

**Nguyên nhân gốc:** thường là một tác vụ nền định kỳ (backup, batch job, garbage collection của một service khác, cron job dọn dẹp) chạy đúng khung giờ đó, tranh chấp tài nguyên I/O hoặc CPU với các container đang phục vụ traffic thật — vấn đề tương tự D3 nhưng có tính chu kỳ rõ ràng theo thời gian thay vì liên tục.

**Khắc phục:** xác định đúng tác vụ nền gây tranh chấp (đối chiếu `docker events`/`crontab` với đúng khung giờ), sau đó hoặc dời khung giờ chạy tác vụ đó sang lúc traffic thấp hơn, hoặc giới hạn tài nguyên của chính tác vụ nền đó (`--cpus`, `ionice` nếu chạy trực tiếp trên host) để nó không tranh chấp mạnh với traffic thật.

**Bài học rút ra:** một sự cố hiệu năng có tính **chu kỳ** gần như luôn chỉ ra một tác vụ định kỳ (scheduled) là thủ phạm — đối chiếu timeline sự cố với lịch cron/batch job trước khi đi sâu vào phân tích cgroup từng container.

---

## Tổng hợp: Checklist chẩn đoán nhanh khi chưa biết bắt đầu từ đâu

1. `docker ps -a` — container nào không `Up` như kỳ vọng?
2. `docker logs --tail 100 -t <container>` — có lỗi rõ ràng không?
3. `docker inspect --format '{{.State.ExitCode}} {{.State.OOMKilled}}' <container>` — có manh mối exit code/OOM không?
4. `docker stats --no-stream` — có container nào bất thường về tài nguyên không?
5. `dmesg -T | grep -i killed` — kernel có tự kill tiến trình nào gần đây không?
6. `docker network inspect <network>` — các container liên quan có chung network không?
7. `df -h /var/lib/docker` và `docker system df` — có vấn đề dung lượng không?
8. `vmstat 1 5` / `top` — host tổng thể có quá tải không?

Áp dụng đúng thứ tự này cho 20 tình huống thực chiến ở [[../20-Enterprise-Project/README|Module 20]] — dự án cuối khóa sẽ không dẫn sẵn bạn đi qua từng bước như tài liệu này, mà chỉ mô tả triệu chứng và yêu cầu bạn tự chẩn đoán.
