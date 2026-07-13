---
title: "02 - Theory"
module: 8
tags: [docker, sysops-infra, module-08, theory, namespaces, cgroups, overlayfs]
---

# 02 - Lý thuyết: Container thực chất là gì

## 1. Virtual Machine vs Container — tại sao container nhẹ hơn

### 1.1 Cách VM tạo cô lập

Một Virtual Machine mô phỏng (ảo hóa) toàn bộ phần cứng: CPU ảo, RAM ảo, đĩa ảo, NIC ảo. Phần mềm chịu trách nhiệm mô phỏng đó là **Hypervisor** (VMware ESXi, KVM, Hyper-V — Type 1 chạy trực tiếp trên phần cứng; VirtualBox — Type 2 chạy trên một Host OS có sẵn).

Vì phần cứng được ảo hóa toàn bộ, mỗi VM cần **một Guest OS đầy đủ** (kernel riêng, driver riêng, tiến trình init riêng — `systemd`, `/sbin/init`...). Khi bạn boot một VM Ubuntu, kernel Linux của VM đó thực sự khởi động từ đầu, y hệt như boot một máy vật lý.

Hệ quả:
- VM tốn vài chục giây đến vài phút để boot (phải qua toàn bộ quy trình boot kernel).
- Mỗi VM tốn RAM/disk cố định cho chính kernel và OS của nó (vài trăm MB đến vài GB) trước khi ứng dụng chạy bất kỳ dòng code nào.
- Cô lập cực mạnh: một VM bị chiếm quyền (compromise) gần như không thể chạm tới kernel của Hypervisor hay VM khác, trừ khi có lỗ hổng nghiêm trọng ở tầng Hypervisor (VM escape — rất hiếm và bị coi là sự cố bảo mật nghiêm trọng).

### 1.2 Cách Container tạo cô lập

Container **không ảo hóa phần cứng và không có kernel riêng**. Tất cả container trên một host **dùng chung một kernel Linux của Host OS**. Thứ làm cho mỗi container "cảm giác" như một máy riêng biệt là hai cơ chế của kernel:

- **Namespaces**: giới hạn những gì một nhóm tiến trình **nhìn thấy** (PID riêng, network stack riêng, hostname riêng, mount point riêng...).
- **Cgroups**: giới hạn những gì một nhóm tiến trình **được phép dùng** (bao nhiêu CPU, bao nhiêu RAM, bao nhiêu I/O).

Vì không có kernel riêng, không có quá trình boot OS, một container chỉ là **một tiến trình Linux bình thường** (hoặc một nhóm tiến trình) được kernel "gắn nhãn" cách ly bằng namespace và giới hạn bằng cgroup. Khởi động container = khởi động tiến trình đó = tính bằng mili-giây, không phải phút.

### 1.3 Bảng so sánh trực diện

| Tiêu chí | Virtual Machine | Container |
|---|---|---|
| Kernel | Mỗi VM có kernel riêng | Dùng chung kernel Host OS |
| Thời gian khởi động | Vài chục giây - vài phút (boot OS đầy đủ) | Mili-giây - vài giây (chỉ khởi động 1 process) |
| Dung lượng | Hàng GB (chứa cả OS) | Thường vài MB - vài trăm MB (chỉ chứa app + dependency) |
| Mật độ trên 1 host | Vài chục VM là đã nặng | Hàng trăm - hàng nghìn container |
| Cô lập | Rất mạnh (ranh giới phần cứng ảo hóa) | Yếu hơn (ranh giới là kernel API - namespace/cgroup) |
| Portable | Đóng gói cả OS, nặng nề khi di chuyển | Đóng gói app + lib, nhẹ, di chuyển nhanh giữa các host cùng kernel Linux |
| Chạy khác OS trên cùng host | Được (Linux VM và Windows VM chung 1 host) | KHÔNG - container Linux cần kernel Linux của host |

Đây chính là câu trả lời cho câu hỏi phỏng vấn "container có phải VM nhẹ hơn không": **không phải**. VM ảo hóa phần cứng, container chia sẻ kernel và cô lập bằng namespace/cgroup — hai cơ chế khác bản chất, không phải cùng công nghệ ở mức độ nhẹ/nặng khác nhau.

### 1.4 Hệ quả thực tế trong vận hành doanh nghiệp

- Một lỗ hổng leo thang đặc quyền trong kernel (kernel privilege escalation CVE) ảnh hưởng đến **toàn bộ container trên host**, vì chúng dùng chung kernel. Đây là lý do vá kernel host và không chạy container với quyền `--privileged` tùy tiện là yêu cầu bảo mật bắt buộc trong doanh nghiệp.
- Vì không cần boot OS, container phù hợp cho kiến trúc scale nhanh (auto-scaling theo traffic, CI/CD build hàng trăm job/ngày) — điều VM khó làm hiệu quả về chi phí và tốc độ.
- VM vẫn có chỗ đứng: khi cần cô lập bảo mật tuyệt đối giữa các tenant (multi-tenant hosting), hoặc khi cần chạy hệ điều hành khác Linux, VM (hoặc container thế hệ mới như Kata Containers/gVisor bổ sung lớp ảo hóa) vẫn là lựa chọn đúng.

## 2. OCI - Open Container Initiative

### 2.1 Tại sao ngành container cần một chuẩn chung

Trước 2015, Docker gần như một mình một chợ, định nghĩa cả format image lẫn cách chạy container. Vấn đề: nếu chỉ một công ty kiểm soát định dạng, hệ sinh thái (registry, orchestrator như Kubernetes, runtime thay thế như rkt) bị phụ thuộc vào quyết định của riêng Docker Inc.

Năm 2015, **Open Container Initiative (OCI)** ra đời dưới Linux Foundation, với sự tham gia của Docker, Google, Red Hat, CoreOS, Microsoft... để chuẩn hóa. Mục tiêu: bất kỳ image nào tuân theo chuẩn OCI đều chạy được trên bất kỳ runtime nào tuân theo chuẩn OCI — không khóa chặt vào một vendor.

### 2.2 Ba spec chính của OCI

1. **Runtime Specification (runtime-spec)**: định nghĩa cách chạy một "filesystem bundle" đã giải nén trên đĩa — gồm file cấu hình `config.json` (chỉ định lệnh chạy, namespace nào cần tạo, cgroup nào áp dụng, mount nào cần gắn, hook nào chạy trước/sau...) và thư mục root filesystem. `runc` là implementation tham chiếu (reference implementation) phổ biến nhất của Runtime Spec.
2. **Image Specification (image-spec)**: định nghĩa cấu trúc một container image — manifest, config, các layer (mỗi layer là một tarball), cách các layer xếp chồng lên nhau. Định dạng image Docker tạo ra khi bạn `docker build` tuân theo chuẩn này.
3. **Distribution Specification (distribution-spec)**: định nghĩa API chuẩn để push/pull image giữa client và registry (ví dụ Docker Hub, Harbor, AWS ECR, Google Artifact Registry). Nhờ chuẩn này, `docker push`/`docker pull` hoạt động được với bất kỳ registry nào tuân thủ, không chỉ Docker Hub.

### 2.3 Luồng OCI trong thực tế

Ở mức cao: một OCI Image được tải về (theo Distribution Spec) → giải nén thành một OCI Runtime filesystem bundle (theo Image Spec) → bundle đó được một OCI Runtime như `runc` chạy lên thành container (theo Runtime Spec). Docker, containerd, Podman, CRI-O đều là các công cụ xây dựng xung quanh luồng chuẩn này — khác nhau ở trải nghiệm người dùng và kiến trúc nội bộ, nhưng đáy cùng đều gọi tới một OCI runtime để thực sự tạo container.

## 3. Namespaces — cơ chế cô lập "nhìn thấy cái gì"

### 3.1 Khái niệm

Namespace là tính năng của kernel Linux giới hạn một nhóm tiến trình chỉ **nhìn thấy một view riêng** của một loại tài nguyên hệ thống, thay vì nhìn thấy toàn bộ. Ví dụ: PID namespace làm cho tiến trình bên trong chỉ thấy các PID trong chính namespace đó, không thấy PID của tiến trình host hay namespace khác.

Namespace được tạo bằng system call `clone()` (khi tạo tiến trình mới kèm cờ `CLONE_NEW*`), `unshare()` (tách tiến trình hiện tại ra khỏi namespace hiện tại, tạo namespace mới), hoặc `setns()` (gia nhập vào một namespace đã tồn tại — đây chính là cơ chế `docker exec` dùng để "chui vào" container đang chạy).

Kernel Linux hiện có (tính đến các bản kernel hiện đại) 8 loại namespace: `mnt`, `pid`, `net`, `ipc`, `uts`, `user`, `cgroup`, và `time` (namespace mới nhất, cô lập đồng hồ boot-time/monotonic). Module này tập trung 6 loại quan trọng nhất với vận hành container hàng ngày.

### 3.2 Từng loại namespace

**PID Namespace (`CLONE_NEWPID`)**
Cô lập cây process ID. Tiến trình đầu tiên tạo ra bên trong một PID namespace mới sẽ có PID = 1 **bên trong namespace đó**, dù trên host nó có thể mang PID = 1651 chẳng hạn. Tiến trình PID 1 bên trong container đóng vai trò như `init`: nó phải reap (dọn) các zombie process con, và nếu nó chết, toàn bộ namespace (và container) bị kernel dọn theo. Đây là lý do vì sao chạy ứng dụng không xử lý signal đúng cách làm PID 1 trong container thường gây ra vấn đề "container không tắt được bằng `docker stop`" (không nhận SIGTERM đúng cách).

**Network Namespace (`CLONE_NEWNET`)**
Cô lập toàn bộ network stack: interface, địa chỉ IP, routing table, iptables rules, socket. Container có network namespace riêng nên có "IP riêng", "loopback riêng" (`127.0.0.1` bên trong container khác với `127.0.0.1` của host). Docker tạo ra một cặp `veth` (virtual ethernet) nối network namespace của container với bridge `docker0` trên host để container có thể ra ngoài.

**Mount Namespace (`CLONE_NEWNS`)**
Cô lập cây mount point. Tiến trình bên trong namespace có view filesystem riêng — mount/umount bên trong không ảnh hưởng ra host và ngược lại (trừ khi cấu hình mount propagation chia sẻ). Đây là namespace lâu đời nhất (namespace đầu tiên được thêm vào Linux, từ đó có tên là `NEWNS` dù về sau còn nhiều namespace khác). Kết hợp với `chroot`/`pivot_root`, mount namespace là nền tảng để container có root filesystem riêng biệt.

**UTS Namespace (`CLONE_NEWUTS`)**
Tên gọi từ "UNIX Time-sharing System". Cô lập hostname và domain name (NIS). Đây là lý do mỗi container có thể có hostname riêng (thường là container ID rút gọn) mà không ảnh hưởng hostname thật của host.

**IPC Namespace (`CLONE_NEWIPC`)**
Cô lập cơ chế Inter-Process Communication kiểu System V (message queue, semaphore, shared memory) và POSIX message queue. Tiến trình trong IPC namespace khác nhau không thể dùng chung shared memory segment hay semaphore trực tiếp — tránh xung đột ID giữa các container không liên quan.

**User Namespace (`CLONE_NEWUSER`)**
Cô lập UID/GID. Đây là namespace **quan trọng nhất về bảo mật** nhưng thường bị bỏ qua trong triển khai Docker mặc định. User namespace cho phép ánh xạ (map) UID 0 (root) **bên trong container** thành một UID không đặc quyền (ví dụ 100000) **trên host**. Nếu tiến trình bên trong container thoát ra khỏi cô lập (container escape) do lỗi cấu hình khác, nó vẫn chỉ là một UID thường trên host, không phải root thật. Docker hỗ trợ tính năng này qua `userns-remap`, nhưng **không bật mặc định** vì tương thích ngược với nhiều workload — đây là điểm hardening quan trọng cần bật thủ công trong môi trường yêu cầu bảo mật cao.

### 3.3 Internal Working: `/proc/<pid>/ns/`

Mỗi tiến trình trên Linux có một thư mục `/proc/<pid>/ns/` chứa các file tượng trưng (symlink) cho từng namespace mà tiến trình đó thuộc về, ví dụ:

```
pid -> pid:[4026532483]
net -> net:[4026532488]
mnt -> mnt:[4026532481]
uts -> uts:[4026532482]
ipc -> ipc:[4026532484]
user -> user:[4026531837]
```

Con số trong ngoặc vuông là **inode number** của namespace đó. Đây là cách chính xác nhất để kiểm tra hai tiến trình có nằm chung một namespace hay không: so sánh inode number. Nếu tiến trình A và B có cùng inode `pid:[...]`, chúng chia sẻ chung PID namespace. Docker dùng chính cơ chế này: khi bạn `docker exec` vào container, Docker gọi `setns()` để tiến trình mới gia nhập đúng các namespace (inode) mà container đó đang sở hữu.

## 4. Cgroups v1 vs v2 — cơ chế giới hạn "được dùng bao nhiêu"

### 4.1 Khái niệm

Control Groups (cgroups) là tính năng kernel Linux tổ chức tiến trình thành các nhóm phân cấp (hierarchy) và áp giới hạn/đo lường/ưu tiên tài nguyên (CPU, RAM, I/O, số lượng PID...) cho từng nhóm. Nếu namespace trả lời câu hỏi "container nhìn thấy gì", cgroup trả lời câu hỏi "container được dùng bao nhiêu tài nguyên của host".

### 4.2 Kiến trúc cây và controller

Cgroup được tổ chức thành cây thư mục ảo, mount tại `/sys/fs/cgroup`. Mỗi node trong cây là một cgroup, có thể chứa cgroup con (kế thừa và tinh chỉnh giới hạn từ cha), và mỗi cgroup gắn với một hoặc nhiều **controller** (còn gọi subsystem): `cpu`, `memory`, `io` (trước đây gọi `blkio`), `pids`, `cpuset`, `hugetlb`...

### 4.3 Khác biệt cgroups v1 và v2

**cgroups v1**: mỗi controller có **cây phân cấp riêng biệt**, mount riêng (`/sys/fs/cgroup/cpu`, `/sys/fs/cgroup/memory`, `/sys/fs/cgroup/blkio`...). Một tiến trình có thể nằm ở vị trí khác nhau trong mỗi cây controller — linh hoạt nhưng phức tạp, khó quản lý nhất quán, và một số controller (đặc biệt `memory` và `io`) khó phối hợp giới hạn chính xác khi cây không đồng nhất.

**cgroups v2**: thống nhất thành **một cây phân cấp duy nhất** (`unified hierarchy`), mount một lần tại `/sys/fs/cgroup`, mọi controller được gắn (attach) vào cùng cây đó qua file `cgroup.controllers` và `cgroup.subtree_control`. Đơn giản hóa quản lý, cho phép giới hạn tài nguyên phối hợp chính xác hơn (ví dụ Pressure Stall Information - PSI để đo mức độ tiến trình bị nghẽn tài nguyên), và là kiến trúc được cộng đồng kernel tích cực phát triển tiếp — cgroups v1 ở trạng thái bảo trì (maintenance mode), không phát triển tính năng mới.

cgroups v2 là **mặc định trên các distro Linux hiện đại** có `systemd` phiên bản từ 247 trở lên (Ubuntu 22.04+, Debian 11+, RHEL 9+, Fedora gần đây). Docker Engine hiện tại hỗ trợ đầy đủ và khuyến nghị cgroups v2.

### 4.4 Cgroup Driver: `systemd` vs `cgroupfs`

Docker (qua containerd/runc) cần một "driver" để thao tác với cgroup filesystem. Có hai lựa chọn:

- **`cgroupfs`**: Docker/runc tự thao tác trực tiếp lên các file trong `/sys/fs/cgroup`, không thông qua `systemd`.
- **`systemd`**: Docker/runc giao việc quản lý cgroup cho `systemd` (vì trên các distro hiện đại, `systemd` là tiến trình PID 1, và chính nó cũng là bộ quản lý cgroup chính của toàn hệ thống thông qua `systemd-cgls`, các unit, slice...).

**Vấn đề khi hai bộ quản lý cgroup cùng tồn tại**: nếu Docker dùng `cgroupfs` trong khi hệ thống dùng `systemd` làm init, bạn có **hai "người quản lý" cùng thao tác một cây cgroup** — dễ dẫn tới trạng thái không nhất quán, đặc biệt khi hệ thống dưới áp lực tài nguyên (memory pressure) khiến `systemd` reload cấu hình cgroup và "giẫm chân" lên cấu hình Docker đã set. Đây là lý do **khuyến nghị bắt buộc trong môi trường production hiện đại là cấu hình Docker dùng cgroup driver `systemd`**, đồng bộ với init system, tránh xung đột. Kubernetes (kubelet) từ các phiên bản gần đây cũng khuyến nghị và dần mặc định driver `systemd` cho cùng lý do.

## 5. OverlayFS — filesystem xếp lớp của container

### 5.1 Vì sao cần một filesystem đặc biệt

Nếu mỗi container clone toàn bộ root filesystem của image, việc tạo container sẽ chậm (copy hàng trăm MB dữ liệu) và tốn disk khủng khiếp (nhân bản dữ liệu giống hệt nhau hàng trăm lần cho hàng trăm container cùng base image). OverlayFS giải quyết bài toán này bằng kỹ thuật **union mount** kết hợp **copy-on-write (CoW)**.

### 5.2 Bốn thành phần cốt lõi

- **`lowerdir`**: một hoặc nhiều thư mục **chỉ đọc** (read-only) — đây chính là các layer của image. Có thể xếp chồng nhiều `lowerdir`.
- **`upperdir`**: thư mục **đọc-ghi** (read-write) — đây là layer ghi duy nhất của container đang chạy, còn gọi container layer. Mọi thay đổi (tạo file mới, sửa file, xóa file) khi container đang chạy đều xảy ra ở đây.
- **`merged`**: view hợp nhất mà tiến trình bên trong container thực sự nhìn thấy và thao tác — kết quả xếp chồng toàn bộ `lowerdir` + `upperdir` lại thành một cây thư mục duy nhất.
- **`workdir`**: thư mục làm việc nội bộ OverlayFS dùng để chuẩn bị file trước khi "chuyển" (atomic move) vào `upperdir` — bắt buộc phải có, cùng filesystem với `upperdir`, và không được truy cập trực tiếp.

### 5.3 Copy-on-Write hoạt động ra sao

Khi tiến trình trong container **đọc** một file chỉ tồn tại ở `lowerdir`, OverlayFS phục vụ trực tiếp từ `lowerdir` — không copy gì cả.

Khi tiến trình **ghi/sửa** một file đang nằm ở `lowerdir`, OverlayFS thực hiện thao tác gọi là **copy-up**: copy toàn bộ file đó từ `lowerdir` lên `upperdir` trước, rồi mới áp thay đổi lên bản copy đó tại `upperdir`. File gốc ở `lowerdir` không bao giờ bị động tới — các layer image luôn bất biến (immutable).

Khi tiến trình **xóa** một file chỉ tồn tại ở `lowerdir` (không có ở `upperdir`), OverlayFS không thể xóa thật (vì `lowerdir` read-only), nên nó tạo một **whiteout file** (file đặc biệt đánh dấu "file này bị xóa") tại `upperdir`. Khi trình bày view `merged`, OverlayFS thấy whiteout thì ẩn file gốc đi, tạo cảm giác file đã bị xóa dù thực ra vẫn còn nguyên trong layer image bên dưới.

### 5.4 Vì sao đây là thiết kế hiệu quả

Vì copy chỉ xảy ra khi ghi (copy-on-write), và chỉ copy đúng file bị sửa (không copy cả layer), container khởi động gần như tức thì (không cần copy gì ở bước khởi tạo), và hàng trăm container dùng chung một base image (ví dụ `python:3.12-slim`) chỉ tốn dung lượng disk cho các layer đó **một lần duy nhất** trên host — mỗi container chỉ tốn thêm dung lượng cho phần `upperdir` (thường rất nhỏ nếu ứng dụng ít ghi dữ liệu vào filesystem của container).

## 6. Image Layer — vì sao image có nhiều lớp

### 6.1 Layer là gì

Mỗi dòng lệnh tạo ra thay đổi filesystem trong Dockerfile (`RUN`, `COPY`, `ADD`) tạo ra **một layer mới** — về bản chất là một bản snapshot các thay đổi filesystem (dưới dạng tarball) so với layer liền trước. Khi build image, các layer được xếp chồng lên nhau đúng theo cơ chế `lowerdir` của OverlayFS đã mô tả ở trên.

Ví dụ Dockerfile:
```dockerfile
FROM python:3.12-slim      # layer 1 (thực ra base image này đã gồm nhiều layer)
RUN pip install flask      # layer 2 - thêm các file gói flask
COPY app.py /app/app.py    # layer 3 - thêm 1 file app.py
CMD ["python", "/app/app.py"]  # không tạo layer filesystem, chỉ là metadata
```

### 6.2 Vì sao thiết kế nhiều layer lại thông minh

- **Cache khi build**: Docker cache từng layer theo hash nội dung. Nếu bạn chỉ sửa `app.py` và build lại, Docker tái sử dụng layer 1 và layer 2 đã cache, chỉ build lại layer 3 — tiết kiệm thời gian build đáng kể trong CI/CD.
- **Chia sẻ giữa nhiều image**: nếu 50 image khác nhau trong công ty đều `FROM python:3.12-slim`, layer của base image đó **chỉ tải về và lưu trên disk một lần**, được tất cả 50 image tham chiếu tới — đúng nguyên lý content-addressable storage bên dưới.
- **Truyền tải hiệu quả (push/pull)**: khi pull một image mới nhưng vài layer đã tồn tại sẵn trên máy (từ image khác cùng base), Docker chỉ tải các layer còn thiếu.

### 6.3 Content-Addressable Storage

Mỗi layer được định danh bằng **hash SHA-256 của chính nội dung nó** (digest), không phải bằng tên do người đặt. Đây gọi là content-addressable storage: địa chỉ lưu trữ (ID) sinh ra trực tiếp từ nội dung. Hệ quả kỹ thuật quan trọng:

- Hai layer có nội dung giống hệt nhau (dù build từ Dockerfile khác nhau, ở thời điểm khác nhau) sẽ có **cùng một digest**, và registry/Docker engine chỉ cần lưu một bản vật lý duy nhất — đây là cơ chế **deduplication** tự nhiên.
- Digest cho phép xác minh tính toàn vẹn: nếu nội dung layer bị thay đổi (dù chỉ 1 byte) trong quá trình truyền tải hay lưu trữ, hash sẽ khác đi, registry/client phát hiện ngay đó là dữ liệu hỏng hoặc bị can thiệp — đây cũng là nền tảng bảo mật của image signing/verification sau này (ví dụ Docker Content Trust, Sigstore/cosign).
- Digest của toàn bộ image (image ID/manifest digest) cũng được tính từ nội dung, nên `image:tag` (ví dụ `nginx:1.27`) chỉ là một **con trỏ có thể thay đổi** trỏ tới một digest cụ thể tại một thời điểm — đây là lý do trong môi trường production nghiêm ngặt, người ta pin theo digest (`nginx@sha256:...`) thay vì tag, để đảm bảo luôn deploy đúng một nội dung bất biến.

## Tổng hợp mối liên hệ

Namespace, cgroup, OverlayFS, và Image Layer không phải 4 khái niệm rời rạc — chúng là 4 mảnh ghép của cùng một bức tranh:

- **Namespace** quyết định container "nhìn thấy" một thế giới riêng (process, network, filesystem view...).
- **Cgroup** quyết định container được "dùng bao nhiêu" tài nguyên vật lý của host.
- **Image Layer** là cách đóng gói filesystem của ứng dụng thành các lớp bất biến, có thể chia sẻ và cache.
- **OverlayFS** là cơ chế kernel xếp các layer đó (`lowerdir`) chồng lên một layer ghi (`upperdir`) để tạo ra root filesystem (`merged`) mà namespace mount (`mnt` namespace) sẽ trình bày cho tiến trình bên trong container.

Khi bạn gõ `docker run nginx`, Docker Engine (qua containerd và runc) thực hiện gần như đồng thời: tạo các namespace mới, gắn cgroup giới hạn tài nguyên, dựng root filesystem bằng OverlayFS từ các layer của image `nginx`, rồi `exec` tiến trình `nginx` vào bên trong tổ hợp cô lập đó. Toàn bộ những gì diễn ra ở Module 09 trở đi thực chất là lớp vỏ tiện dụng bọc quanh 4 cơ chế kernel này.
