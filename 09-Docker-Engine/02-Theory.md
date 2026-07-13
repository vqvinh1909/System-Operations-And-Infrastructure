---
title: "02 - Theory"
module: 9
tags: [docker, sysops-infra, module-09, theory, internal-working]
---

# 02 - Lý thuyết chi tiết

## 1. Kiến trúc Client-Server của Docker

### Tại sao thiết kế client-server

Docker CLI (`docker`) **không** trực tiếp tạo container. Nó là một client gửi HTTP request tới `dockerd` — server thực sự làm việc. Thiết kế này quan trọng vì:

- **Quản trị từ xa**: bạn có thể chạy `docker` CLI trên máy laptop nhưng điều khiển `dockerd` chạy trên server ở xa, chỉ cần trỏ `DOCKER_HOST` tới địa chỉ daemon đó. Đây là cách nhiều công cụ CI/CD và Ansible tương tác với Docker trên server production mà không cần SSH vào gõ lệnh trực tiếp.
- **Nhiều client cùng dùng chung một daemon**: `docker` CLI, Docker Desktop, Portainer, hay chương trình tự viết dùng Docker SDK — tất cả chỉ là các client khác nhau nói chuyện với cùng một `dockerd` qua cùng một API.
- **Tách quyền hạn**: client có thể chạy với quyền user thường, trong khi chỉ daemon mới cần chạy với quyền root để thao tác namespace/cgroup. (Vấn đề bảo mật phát sinh từ đây sẽ phân tích kỹ ở mục 4.)

### Cách CLI giao tiếp với daemon

Mặc định trên Linux, `dockerd` lắng nghe trên **Unix domain socket** tại `/var/run/docker.sock` (thường là symlink hoặc đường dẫn thực tới `/run/docker.sock`). Khi bạn gõ:

```bash
docker run -d --name web nginx:alpine
```

CLI dịch lệnh này thành một chuỗi HTTP request theo **Docker Engine REST API** (ví dụ `POST /containers/create`, sau đó `POST /containers/{id}/start`), gửi qua socket đó. Vì là Unix socket (không phải TCP), giao tiếp này chỉ diễn ra nội bộ trên cùng một máy, và quyền truy cập được kiểm soát bằng **file permission của chính file socket** — đây là nền tảng của vấn đề bảo mật nhóm `docker` sẽ nói ở phần sau.

Docker daemon cũng có thể được cấu hình lắng nghe qua TCP socket (ví dụ `tcp://0.0.0.0:2375`) để quản trị từ xa, nhưng **mặc định không bật TLS** trên cổng này — đây là một cấu hình cực kỳ nguy hiểm nếu bật không kiểm soát trong môi trường doanh nghiệp, vì ai kết nối được tới cổng đó coi như có toàn quyền root trên host (chi tiết ở [[06-Troubleshooting]]).

## 2. Bốn thành phần cốt lõi và luồng gọi

Xem sơ đồ đầy đủ tại [[03-Architecture]]. Ở đây ta phân tích vai trò và cơ chế bên trong từng thành phần.

### 2.1 dockerd (Docker Daemon)

`dockerd` là tiến trình nền chạy với quyền root (thường được khởi động và quản lý bởi systemd, unit `docker.service`). Trách nhiệm của `dockerd`:

- Cung cấp REST API cho client.
- Quản lý **image**: pull, push, build, tag, xóa (build thực tế được ủy quyền cho BuildKit — builder mặc định từ Docker Engine 23.0 trở đi).
- Quản lý **network**: tạo bridge network, cấu hình iptables/nftables cho NAT và port mapping.
- Quản lý **volume**: tạo, gắn, xóa volume.
- Ủy quyền việc quản lý vòng đời container xuống cho `containerd` qua gRPC API (thường qua socket `/run/containerd/containerd.sock`).

**Internal Working**: `dockerd` không tự mình gọi `clone()`/`unshare()` để tạo namespace. Nó dịch yêu cầu của người dùng (image nào, mount gì, network gì, giới hạn tài nguyên bao nhiêu) thành một **OCI-compliant container spec** (file `config.json` theo chuẩn OCI Runtime Spec — hiện tại v1.3.0), rồi gửi spec đó cho `containerd` xử lý.

### 2.2 containerd

`containerd` là một **container runtime ở tầng cao (high-level runtime)**, khác với `runc` là **runtime ở tầng thấp (low-level runtime)**. Trách nhiệm:

- Quản lý vòng đời container: create, start, stop, delete — theo dõi trạng thái từng container.
- Quản lý content store: layer image, snapshot filesystem (làm việc trực tiếp với snapshotter, ví dụ overlayfs snapshotter — liên hệ Module 08 về OverlayFS).
- Quản lý namespace logic của riêng nó (khác namespace kernel Linux — đây là namespace ở tầng containerd để cô lập nhiều client dùng chung một containerd, ví dụ Docker dùng namespace `moby`, còn Kubernetes qua CRI dùng namespace `k8s.io`).

**Điểm mấu chốt cần nhớ**: `containerd` **độc lập với `dockerd`**. Bạn có thể cài `containerd` và dùng công cụ `ctr` hoặc `nerdctl` để chạy container mà không cần `dockerd` chạy. Đây chính là cách nhiều cụm Kubernetes hiện đại vận hành — chúng nói chuyện thẳng với `containerd` qua CRI, không đi qua `dockerd`.

### 2.3 containerd-shim

Đây là thành phần hay bị bỏ qua nhưng lại giải thích được nhiều hành vi khó hiểu khi vận hành thực tế. Với mỗi container đang chạy, `containerd` sẽ khởi tạo (fork/exec) **một tiến trình shim riêng** (thường tên là `containerd-shim-runc-v2`).

**Tại sao cần shim — hai lý do cốt lõi:**

1. **Tránh orphan process khi containerd/dockerd restart hoặc crash**: `runc` chỉ được dùng để **tạo** container rồi **thoát ngay** (runc không ở lại làm tiến trình cha của container). Tiến trình cha thực sự giữ container process chính là `containerd-shim`. Vì shim là một tiến trình độc lập, tách rời khỏi `containerd`, nên khi bạn `systemctl restart containerd` hoặc thậm chí `systemctl restart docker`, các container **vẫn tiếp tục chạy bình thường** — shim vẫn sống, vẫn giữ container process, không có gì bị "mồ côi" hay bị giết theo.
2. **containerd không cần giữ một process cha riêng cho từng container**: nếu `containerd` tự làm cha trực tiếp của hàng trăm container, việc thu thập exit status (reap zombie process qua `wait()`), truyền tín hiệu, quản lý I/O (stdin/stdout/stderr) sẽ dồn hết logic và rủi ro vào một tiến trình duy nhất. Thay vào đó, mỗi container có shim riêng lo việc đó, và `containerd` chỉ cần giao tiếp với shim qua một giao thức nhẹ để hỏi trạng thái, đọc log, chuyển tín hiệu.

**Internal Working**: shim đảm nhận việc gọi `runc create`/`runc start`, sau đó `runc` thoát, còn shim tiếp tục sống làm cha của tiến trình container (qua kỹ thuật reparenting của Linux — khi tiến trình cha trực tiếp thoát, tiến trình con được `init`/`subreaper` nhận nuôi; shim tự đăng ký làm subreaper bằng `prctl(PR_SET_CHILD_SUBREAPER)` để đảm bảo chính nó, chứ không phải PID 1 của host, nhận nuôi container process). Khi container thoát, shim ghi lại exit code, giữ lại để `containerd`/`dockerd` đọc (đây là lý do bạn vẫn xem được exit code bằng `docker inspect` dù container đã dừng từ lâu), rồi mới tự thoát.

### 2.4 runc (OCI Runtime)

`runc` là công cụ dòng lệnh triển khai chuẩn **OCI Runtime Specification**. Nhiệm vụ duy nhất của nó: đọc file `config.json` (OCI spec) mô tả container cần tạo (namespace nào, cgroup giới hạn ra sao, rootfs ở đâu, entrypoint là gì), rồi gọi các system call kernel cần thiết:

- `clone()`/`unshare()` với các flag namespace (`CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWNS`, `CLONE_NEWUTS`, `CLONE_NEWIPC`, `CLONE_NEWUSER`...) — chính là những namespace bạn đã học ở Module 08.
- Thiết lập cgroup (v2, driver `systemd` được khuyến nghị trên các distro hiện đại thay vì `cgroupfs`) để giới hạn CPU/memory/IO.
- `pivot_root()`/`chroot()` để đổi root filesystem của tiến trình sang rootfs của container (ghép từ các layer image qua OverlayFS).
- Cuối cùng gọi `execve()` để nạp và chạy chương trình entrypoint bên trong các namespace vừa tạo — đây chính là tiến trình PID 1 bên trong container.

Sau khi container process được tạo và chạy, `runc` **thoát ngay lập tức** — nó không phải tiến trình chạy nền. Đây là điểm rất dễ nhầm: nhiều người tưởng `runc` là "runtime chạy container suốt vòng đời", thực ra nó chỉ là công cụ "tạo một lần rồi thoát", tiến trình sống lâu dài giữ container là **shim**, không phải runc.

## 3. Registry: Docker Hub, private registry, và image naming

### Vì sao cần registry

Image có thể lớn hàng trăm MB tới vài GB. Bạn không thể "gửi email" image cho đồng nghiệp hay server production mỗi lần deploy. Registry là kho lưu trữ tập trung để **push** (đẩy image lên) và **pull** (kéo image về) một cách có kiểm soát, có versioning (qua tag), có phân quyền truy cập.

- **Docker Hub**: registry công khai mặc định (`docker.io`), miễn phí lưu trữ image public, có giới hạn rate-limit khi pull ẩn danh — đây là lý do nhiều tổ chức gặp lỗi `429 Too Many Requests` khi CI/CD pull base image quá nhiều lần trong ngày mà không login.
- **Private registry**: doanh nghiệp thường tự host registry nội bộ (ví dụ Harbor, hoặc registry của cloud provider như ECR/ACR/GCR) để: (1) không phụ thuộc internet/Docker Hub khi build/deploy, (2) kiểm soát image nào được phép chạy trong hạ tầng (image scanning, signing), (3) tránh rate-limit, (4) giữ image chứa mã nguồn nội bộ không public ra ngoài.

### Cấu trúc tên image: `registry/repository:tag`

```
registry.company.local:5000/backend-team/payment-api:v2.3.1
└────────────┬────────────┘ └──────┬───────┘ └────┬────┘ └──┬──┘
          registry              namespace     repository    tag
```

- Nếu bỏ qua phần `registry`, Docker mặc định hiểu là Docker Hub (`docker.io`).
- Nếu bỏ qua `tag`, Docker mặc định dùng `latest` — **đây là thói quen nguy hiểm trong production** vì `latest` không đảm bảo tính bất biến (immutable); ai đó push đè `latest` là mọi nơi đang chạy `latest` có thể vô tình pull phải phiên bản khác vào lần deploy tiếp theo. Nguyên tắc thực chiến: production luôn pin tag cụ thể (ví dụ theo version hoặc theo Git SHA), không dùng `latest`.

### Luồng pull/push

**Pull**: `docker pull registry/repo:tag` → `dockerd` gọi API của registry (chuẩn OCI Distribution Spec) → registry trả về manifest (danh sách layer digest cần tải) → `dockerd` kiểm tra layer nào đã có sẵn cục bộ (theo digest, tránh tải trùng) → chỉ tải các layer còn thiếu → ghép layer qua storage driver (OverlayFS2, liên hệ Module 08) thành image hoàn chỉnh.

**Push**: ngược lại — `dockerd` tính digest từng layer, kiểm tra registry đã có layer đó chưa (registry hỗ trợ tránh upload trùng), chỉ đẩy layer mới, cuối cùng đẩy manifest.

## 4. Docker Daemon: cấu hình và quyền truy cập — góc nhìn bảo mật

### File cấu hình daemon.json

`dockerd` đọc cấu hình từ `/etc/docker/daemon.json` (JSON) khi khởi động. Các tùy chọn phổ biến: driver logging mặc định, địa chỉ registry mirror, cgroup driver, giới hạn số file descriptor, DNS mặc định cho container... Chi tiết ví dụ cụ thể xem [[04-Commands]].

Một điểm cần nhớ: **sửa `daemon.json` không tự áp dụng ngay** — phải `systemctl reload docker` (với các thay đổi hỗ trợ reload) hoặc `systemctl restart docker` (với thay đổi cần restart, ví dụ đổi storage driver) để áp dụng. Restart daemon **không** làm container đang chạy bị dừng (nhờ kiến trúc shim đã giải thích ở trên) nhưng sẽ làm gián đoạn khả năng quản lý container tạm thời trong lúc daemon khởi động lại.

### Rủi ro bảo mật: nhóm `docker` = root-equivalent access

Đây là một trong những hiểu lầm bảo mật phổ biến nhất với người mới dùng Docker: nhiều người nghĩ thêm user vào nhóm `docker` chỉ là "cho phép dùng Docker mà không cần gõ sudo", tương tự như nhóm `sudo` hạn chế theo lệnh. **Thực tế không phải vậy.**

`/var/run/docker.sock` có owner `root`, group `docker`, permission mặc định `660` (đọc/ghi cho owner và group, không cho other). Bất kỳ ai là thành viên nhóm `docker` đều kết nối được socket này **mà không cần mật khẩu, không qua sudo, không bị ghi log như sudo**. Và vì `dockerd` chạy với quyền root, mọi request qua socket đều được `dockerd` thực thi **với quyền root trên host**.

Hệ quả: một user trong nhóm `docker`, dù không có quyền `sudo`, vẫn có thể lấy toàn quyền root trên host chỉ bằng một lệnh:

```bash
docker run -v /:/host -it alpine chroot /host /bin/sh
```

Lệnh này mount toàn bộ filesystem của host vào container, rồi `chroot` vào đó — bên trong container này, bạn đang thao tác trực tiếp trên filesystem thật của host với quyền root, có thể đọc `/etc/shadow`, sửa `/etc/sudoers`, cài backdoor, hay đơn giản là đọc mọi secret nằm trên host.

**Ý nghĩa thực chiến cho sysadmin**: khi cấp quyền `docker` cho user, hãy coi đó tương đương cấp quyền `root` (hoặc `sudo ALL=(ALL) NOPASSWD:ALL`), không phải một quyền hạn chế. Trong môi trường doanh nghiệp:

- Hạn chế tối đa số người trong nhóm `docker` trên server production.
- Cân nhắc dùng **rootless Docker** (chạy `dockerd` không cần quyền root, dùng user namespace để map UID) cho các môi trường nhạy cảm — đây là tính năng chính thức của Docker nhằm giảm bề mặt tấn công này.
- Với CI/CD runner cần build image, cân nhắc dùng công cụ build không cần daemon chạy full quyền (ví dụ Kaniko trong Kubernetes) thay vì mount `docker.sock` vào container CI (một anti-pattern rất phổ biến gọi là "Docker-in-Docker qua socket mount" — vô tình cấp quyền root host cho bất kỳ pipeline CI nào chạy trên runner đó).

Xem sơ đồ minh họa luồng rủi ro này ở [[03-Architecture]].

## 5. Container Lifecycle — trạng thái và chuyển trạng thái

Container có một tập trạng thái rõ ràng mà `docker ps -a` hoặc `docker inspect` (trường `State.Status`) thể hiện: `created`, `running`, `paused`, `exited`, `dead` (hiếm gặp, khi daemon không dọn dẹp được container do lỗi). Không có "xóa" là một trạng thái — remove là hành động chuyển container ra khỏi hệ thống hoàn toàn.

### Created → Running

`docker create` chỉ tạo container (cấp container ID, chuẩn bị filesystem, network) nhưng **chưa chạy tiến trình entrypoint**. `docker start` mới thực sự gọi tới `runc start`, chạy tiến trình PID 1 bên trong container. `docker run` = `docker create` + `docker start` gộp lại (kèm theo tùy chọn attach vào stdout nếu không dùng `-d`).

### Running → Paused → Running

`docker pause` dùng **cgroup freezer** (`cgroup.freeze` trong cgroup v2) để tạm dừng toàn bộ tiến trình bên trong container ở trạng thái "đông cứng" — tiến trình vẫn tồn tại trong bộ nhớ, vẫn giữ toàn bộ state, nhưng không được lập lịch chạy trên CPU. `docker unpause` giải đông, tiến trình tiếp tục chạy đúng chỗ nó bị dừng. Khác biệt quan trọng so với dừng hẳn: **không có tín hiệu nào được gửi tới tiến trình**, ứng dụng bên trong hoàn toàn không biết mình vừa bị đóng băng.

Tình huống thực tế dùng `pause`: tạm dừng một container đang debug để chụp lại trạng thái filesystem/tiến trình mà không làm nó tiếp tục ghi log hay xử lý request, hoặc tạm dừng container tiêu tốn CPU trong lúc xử lý sự cố khẩn cấp trên node mà không muốn mất state của nó (khác `stop` sẽ kết thúc hẳn tiến trình).

### Running → Exited: `stop` vs `kill` — khác biệt sống còn khi vận hành

Đây là một trong những khác biệt quan trọng nhất cần nhớ nằm lòng:

**`docker stop`**: gửi tín hiệu **SIGTERM** cho tiến trình PID 1 trong container, sau đó **chờ một khoảng thời gian gia hạn (grace period, mặc định 10 giây)**. Nếu tiến trình tự thoát trong khoảng thời gian đó (vì đã bắt SIGTERM và dọn dẹp xong — đóng kết nối DB, flush buffer, ghi log), container chuyển sang `exited` với đúng exit code của tiến trình đó. Nếu hết grace period mà tiến trình chưa thoát, Docker gửi tiếp **SIGKILL** để buộc dừng ngay lập tức.

**`docker kill`**: gửi tín hiệu **SIGKILL** (hoặc tín hiệu khác nếu chỉ định bằng `--signal`) **ngay lập tức**, không có grace period, không cho tiến trình cơ hội dọn dẹp.

**Tại sao khác biệt này quan trọng trong vận hành thực tế**: một ứng dụng xử lý transaction (ví dụ service ghi dữ liệu vào database) nếu bị `kill` giữa chừng có thể để lại dữ liệu ở trạng thái không nhất quán, connection pool không được đóng sạch, hay file tạm không được dọn. `docker stop` cho ứng dụng cơ hội tắt "graceful" (đúng chuẩn 12-factor app: lắng nghe SIGTERM để dọn dẹp trước khi thoát). Trong quy trình deploy/rolling update chuẩn (kể cả khi Docker Compose hay orchestrator gọi xuống), luôn ưu tiên `stop`. `kill` chỉ nên dùng khi container "treo" không phản hồi SIGTERM (ví dụ tiến trình bị deadlock) và cần giải phóng ngay.

### Exited → Removed

`docker rm` xóa hẳn container khỏi hệ thống: giải phóng container ID, xóa writable layer (layer ghi phía trên các layer image chỉ đọc — liên hệ Module 08), xóa cấu hình network gắn với container đó. Đây là hành động **không thể hoàn tác** — dữ liệu ghi trong writable layer (không dùng volume) sẽ mất vĩnh viễn.

Xem sơ đồ trạng thái đầy đủ (dạng box ASCII) tại [[03-Architecture]].

## 6. Image, Image ID, Tag — liên hệ Module 08

Một **image** là tập hợp các layer chỉ đọc xếp chồng (đã học ở Module 08: mỗi layer là một tập thay đổi filesystem, ghép lại qua OverlayFS) cộng với metadata mô tả cách chạy (entrypoint, cmd, env, exposed port...) dưới dạng **image manifest** tuân theo OCI Image Spec (hiện tại v1.1.0).

- **Image ID**: chuỗi hash (SHA-256) tính từ nội dung config của image — thay đổi bất kỳ layer hay metadata nào cũng ra Image ID khác. Đây là định danh **nội dung** (content-addressable), không phải định danh do người đặt tùy ý.
- **Tag**: nhãn con người đặt (ví dụ `v2.3.1`, `latest`) trỏ tới một Image ID cụ thể tại một thời điểm — tag có thể bị **di chuyển** (push đè) sang Image ID khác, đây là lý do tag không đáng tin bằng digest khi cần tính bất biến tuyệt đối (production pipeline nên pin theo digest `@sha256:...` nếu cần đảm bảo 100% không đổi).

Module 10 sẽ đi sâu vào cách build image, Dockerfile, và tối ưu layer — module này chỉ cần bạn nắm vững khái niệm nền để hiểu lệnh `docker pull`/`docker run` đang thao tác trên cái gì.

---

Tiếp theo: [[03-Architecture]] để xem toàn bộ sơ đồ kiến trúc trực quan hóa những gì vừa học.
