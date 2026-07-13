---
title: "02 - Theory"
module: 11
tags: [docker, sysops-infra, module-11, theory, lifecycle, cgroups, signals, logging]
---

# 02 — Lý thuyết

## 1. Container Lifecycle chi tiết

### 1.1 Các trạng thái (states)

Một container Docker luôn ở một trong các trạng thái sau (bạn có thể xem trạng thái hiện tại bằng `docker ps -a` cột `STATUS`, hoặc chính xác hơn bằng `docker inspect --format '{{.State.Status}}'`):

| Trạng thái | Ý nghĩa |
|---|---|
| `created` | Container đã được tạo (đã cấp phát filesystem, network namespace...) nhưng **tiến trình chính chưa chạy**. |
| `running` | Tiến trình chính (entrypoint/cmd) đang chạy. |
| `paused` | Tiến trình vẫn tồn tại trong bộ nhớ nhưng bị **đóng băng hoàn toàn** — không được cấp CPU time. |
| `restarting` | Container đang trong quá trình khởi động lại (do restart policy hoặc do bạn gọi `docker restart`). |
| `exited` | Tiến trình chính đã kết thúc (thoát bình thường hoặc bị kill), container vẫn còn tồn tại trên đĩa (filesystem, log, cấu hình) — chỉ chưa bị xóa. |
| `dead` | Trạng thái hiếm gặp, khi Docker daemon không thể dọn dẹp container do lỗi (ví dụ mount bị treo) — cần can thiệp thủ công. |
| (removed) | Không còn là một "trạng thái" theo đúng nghĩa — container đã bị xóa hoàn toàn khỏi hệ thống, không còn xuất hiện trong `docker ps -a`. |

### 1.2 Các lệnh chuyển trạng thái và sự khác biệt dễ nhầm nhất

Đây là phần hay bị hiểu sai nhất khi mới học Docker — đặc biệt là khác biệt giữa `docker create`, `docker start`, và `docker run`:

- **`docker create`**: Chỉ tạo container (cấp phát filesystem từ image, gán container ID, tạo network namespace) — **không chạy** tiến trình chính. Trạng thái sau lệnh này: `created`.
- **`docker start`**: Khởi động tiến trình chính của một container **đã tồn tại** (đang ở trạng thái `created` hoặc `exited`). Trạng thái chuyển sang `running`.
- **`docker run`**: Thực chất là **gộp `docker create` + `docker start`** (và mặc định thêm `docker attach` nếu không có `-d`). Đây là lệnh bạn dùng nhiều nhất khi tạo container mới lần đầu.
- **`docker stop`**: Gửi `SIGTERM` cho tiến trình chính (PID 1 trong container), đợi một khoảng thời gian ân hạn (grace period, mặc định 10 giây, chỉnh bằng `-t`/`--time`), nếu tiến trình chưa thoát thì gửi `SIGKILL` để buộc dừng. Trạng thái chuyển sang `exited`.
- **`docker restart`**: Về bản chất là `docker stop` + `docker start` — cũng tuân theo cùng cơ chế grace period ở bước dừng.
- **`docker pause`**: Dùng cgroups freezer để **đóng băng** toàn bộ tiến trình trong container — không cấp CPU time, tiến trình vẫn nằm nguyên trong bộ nhớ, không nhận bất kỳ signal nào (kể cả `SIGKILL` không có tác dụng cho tới khi unpause). Dùng khi cần "tạm ngưng" container mà không muốn mất trạng thái trong RAM (ví dụ dừng tạm một container debug để rảnh tay CPU cho việc khác).
- **`docker unpause`**: Đưa container từ `paused` về lại `running`, tiến trình tiếp tục chạy đúng ngay chỗ nó bị đóng băng.
- **`docker kill`**: Gửi signal **ngay lập tức** cho tiến trình chính — mặc định là `SIGKILL` (dừng cứng, không có grace period). Có thể chỉ định signal khác bằng `--signal` (ví dụ gửi `SIGHUP` để yêu cầu ứng dụng reload cấu hình mà không cần dừng container).
- **`docker rm`**: Xóa hẳn một container đã dừng (`exited`/`created`) khỏi hệ thống — filesystem container, log, cấu hình bị xóa vĩnh viễn. Không xóa được container đang `running` trừ khi thêm `-f` (force, thực chất là `kill` rồi `rm`).

### 1.3 Sự khác biệt cốt lõi: `stop` vs `kill`

Đây là câu hỏi phỏng vấn kinh điển và cũng là nguyên nhân của rất nhiều sự cố mất dữ liệu thật:

| | `docker stop` | `docker kill` |
|---|---|---|
| Signal đầu tiên | `SIGTERM` (yêu cầu dừng nhẹ nhàng) | `SIGKILL` mặc định (buộc dừng ngay, không đàm phán) |
| Grace period | Có (mặc định 10s, chỉnh bằng `-t`) | Không |
| Ứng dụng có cơ hội dọn dẹp không | Có — đóng kết nối DB, flush buffer, ghi nốt file dở | Không — tiến trình bị kernel buộc dừng ngay lập tức |
| Khi nào dùng | Vận hành bình thường: deploy, restart theo kế hoạch | Container treo cứng, không phản hồi `SIGTERM`, cần dừng khẩn cấp |

**Bài học thực chiến:** Nếu ứng dụng của bạn ghi dữ liệu vào file hoặc giữ kết nối database, luôn ưu tiên `docker stop`. Dùng `docker kill` tùy tiện trong production tương đương với việc rút phích cắm điện đột ngột — có thể làm hỏng file đang ghi dở hoặc để lại transaction database ở trạng thái không nhất quán.

## 2. Signal Handling — PID 1 và vấn đề Zombie Process

### 2.1 Vì sao PID 1 trong container "đặc biệt"

Đây là kiến thức Linux bạn đã học ở khóa Sysadmin (signal, process tree) áp dụng trực tiếp vào container. Trong một hệ điều hành Linux thông thường, **PID 1** là `init`/`systemd` — tiến trình đặc biệt có 2 trách nhiệm:

1. **Reap zombie process**: Khi một tiến trình con kết thúc, nó trở thành "zombie" cho tới khi tiến trình cha gọi `wait()` để "thu dọn" (reap) nó. Nếu tiến trình cha đã chết, zombie được PID 1 "nhận nuôi" (re-parent) và reap thay.
2. **Signal forwarding mặc định của kernel cho PID 1 khác biệt**: Kernel Linux **không áp dụng hành vi mặc định (default disposition)** của một số signal (như `SIGTERM`) cho PID 1 **trừ khi tiến trình đó tự đăng ký handler xử lý signal đó**. Đây là quy tắc đặc biệt của kernel dành riêng cho PID 1.

Trong một container, tiến trình `ENTRYPOINT`/`CMD` của bạn **chính là PID 1** của container đó (nhìn từ bên trong PID namespace của container). Đây là điểm khác biệt lớn với chạy ứng dụng trực tiếp trên host — trên host, ứng dụng của bạn là con của `systemd` (PID 1 thật của hệ điều hành) nên không gặp vấn đề này.

### 2.2 Hai hệ quả thực tế

**Hệ quả 1 — Container "không chịu dừng" khi `docker stop`:**

Nếu ứng dụng của bạn (ví dụ một script Python hay Node.js chạy trực tiếp làm PID 1) **không tự đăng ký handler xử lý `SIGTERM`**, kernel sẽ **không áp dụng hành vi mặc định** (vốn dĩ là "thoát tiến trình") cho nó. Kết quả: `docker stop` gửi `SIGTERM`, ứng dụng bỏ qua signal (vì không có handler và kernel không tự xử lý thay), Docker đợi hết grace period (mặc định 10 giây) rồi buộc phải gửi `SIGKILL`. Container "chậm dừng" 10 giây mỗi lần deploy — nhân với hàng trăm lần deploy mỗi tháng, đây là chi phí thời gian thật không hề nhỏ, và tệ hơn là ứng dụng **không có cơ hội graceful shutdown** (không kịp đóng kết nối, flush cache).

**Hệ quả 2 — Zombie process tích lũy:**

Nếu ứng dụng của bạn là PID 1 và tự spawn ra các tiến trình con (ví dụ một shell script gọi nhiều lệnh con, hoặc một ứng dụng fork worker process), nhưng **không tự implement logic reap zombie** (hầu hết ứng dụng thông thường không làm việc này — đó là việc của `init`, không phải việc của business logic), các tiến trình con khi kết thúc sẽ trở thành zombie **vĩnh viễn không ai reap** — vì PID 1 (ứng dụng của bạn) không biết/không làm nhiệm vụ reap của init. Zombie tích lũy dần theo thời gian, chiếm slot trong bảng tiến trình (process table) của kernel — dù không tốn CPU/RAM đáng kể, nhưng khi bảng tiến trình đầy, container **không thể fork thêm tiến trình mới**.

### 2.3 Giải pháp: init process nhẹ (`--init` hoặc `tini`)

Docker cung cấp cờ `--init` (từ `docker run --init ...`) để chèn một **init process cực nhẹ** (Docker dùng `tini` bên dưới) làm PID 1 thật sự trong container. Init process này làm đúng 2 việc:

1. Nhận signal (`SIGTERM`, `SIGINT`...) và **forward đúng signal đó** xuống cho tiến trình ứng dụng thật (giờ là PID con, không phải PID 1 nữa) — tiến trình ứng dụng dù không tự đăng ký handler đặc biệt vẫn nhận được hành vi signal đúng như khi chạy bình thường trên host.
2. Tự động reap mọi zombie process con.

```
Không có --init:                       Có --init:

PID 1 = ứng dụng của bạn                PID 1 = tini (init nhẹ)
   |                                       |
   |  SIGTERM có thể bị bỏ qua             |  nhận SIGTERM, forward xuống
   |  (nếu không tự đăng ký handler)       v
   v                                    PID con = ứng dụng của bạn
(không reap zombie con)                    |
                                            |  ứng dụng xử lý signal
                                            |  bình thường như trên host
                                        tini tự reap zombie
```

**Khuyến nghị thực chiến:** Với ứng dụng đơn giản (một tiến trình duy nhất, không fork con) và bản thân framework đã tự xử lý `SIGTERM` tốt (nhiều framework hiện đại làm điều này), không bắt buộc phải dùng `--init`. Nhưng với bất kỳ container nào chạy shell script làm entrypoint, hoặc ứng dụng có khả năng spawn tiến trình con, **luôn bật `--init`** — chi phí gần như bằng 0, lợi ích tránh được lớp sự cố khó debug nhất (container "không chịu chết đúng cách", hoặc process table đầy sau vài tuần chạy).

## 3. Resource Limit và cgroups v2

### 3.1 Nhắc lại cgroups v2 (liên hệ Module 08 Linux Sysadmin)

Bạn đã học ở Linux Sysadmin: cgroups (control groups) là cơ chế kernel Linux dùng để **giới hạn, đo lường, và cô lập** tài nguyên (CPU, memory, I/O...) cho một nhóm tiến trình. Docker **không tự phát minh** cơ chế giới hạn tài nguyên — nó chỉ là lớp CLI/API tiện lợi, bên dưới gọi thẳng xuống cgroups v2 mà kernel cung cấp sẵn. Trên các distro Linux hiện đại (systemd ≥ 247), cgroups v2 là mặc định, và driver cgroup khuyến nghị là `systemd` (thay vì `cgroupfs` kiểu cũ) để tránh xung đột giữa `systemd` và Docker khi cùng quản lý cgroup.

Khi bạn chạy `docker run --memory=512m ...`, Docker daemon thực chất tạo một cgroup con cho container đó và ghi giá trị vào file ảo trong `/sys/fs/cgroup/...` tương ứng — đúng thao tác bạn đã từng làm thủ công bằng tay ở Module 08.

### 3.2 Bảng mapping flag Docker sang cgroups v2

| Flag Docker | Ý nghĩa | File cgroups v2 tương ứng |
|---|---|---|
| `--memory` (`-m`) | Giới hạn cứng lượng RAM tối đa container được dùng | `memory.max` |
| `--memory-reservation` | Ngưỡng "mềm" — kernel cố gắng giữ container dưới mức này khi hệ thống có áp lực bộ nhớ, nhưng không cứng như `--memory` | `memory.low` |
| `--memory-swap` | Tổng giới hạn RAM + swap. Nếu bằng giá trị `--memory`, container **không được dùng swap** | Kết hợp `memory.max` và `memory.swap.max` |
| `--cpus` | Số lượng CPU (có thể là số thập phân, ví dụ `1.5`) container được dùng tối đa | `cpu.max` (quota/period) |
| `--cpu-shares` | Trọng số CPU **tương đối** khi có tranh chấp — không giới hạn cứng | `cpu.weight` |
| `--cpuset-cpus` | Ghim (pin) container chỉ được chạy trên các CPU core cụ thể (ví dụ `0,1`) | `cpuset.cpus` |

**Lưu ý quan trọng về cơ chế CPU weight-based:** Với cgroups v2, `--cpus` không "khóa cứng" container vào đúng số CPU đó theo kiểu phần cứng — nó hoạt động qua cơ chế **quota/period**: kernel chia thời gian thành các chu kỳ nhỏ (period, mặc định 100ms), và giới hạn tổng thời gian CPU container được dùng trong mỗi chu kỳ đó (quota) tương ứng với giá trị `--cpus` bạn đặt. Ví dụ `--cpus=1.5` nghĩa là container được dùng tối đa 150ms CPU-time trong mỗi chu kỳ 100ms (tương đương 1.5 lõi CPU đầy).

### 3.3 Điều gì xảy ra khi vượt giới hạn Memory — OOM Kill

Đây là phần quan trọng nhất về mặt vận hành. Khi container cố gắng cấp phát bộ nhớ vượt quá `memory.max` (tức `--memory` bạn đặt):

1. Kernel Linux kích hoạt **OOM Killer** (Out-Of-Memory Killer) — cùng cơ chế bạn đã học ở Linux Sysadmin khi hệ thống cạn RAM, nhưng lần này **phạm vi bị giới hạn trong cgroup của container đó**, không ảnh hưởng tới toàn bộ host (đây chính là giá trị của việc set `--memory` — cô lập sự cố OOM vào đúng một container, không để nó kéo sập cả server).
2. OOM Killer chọn tiến trình để kill (thường dựa trên điểm `oom_score`) — trong hầu hết trường hợp là tiến trình chính (PID 1) của container.
3. Tiến trình bị kill bằng `SIGKILL` — không có cơ hội graceful shutdown, không flush được buffer, không đóng transaction đang dở.
4. Container chuyển sang trạng thái `exited` với **exit code 137** (137 = 128 + 9, trong đó 9 là số hiệu của `SIGKILL`).
5. Nếu có restart policy phù hợp (`on-failure` hoặc `always`), Docker sẽ khởi động lại container — và nếu nguyên nhân gốc (workload thật sự cần nhiều RAM hơn, hoặc memory leak trong code) không được xử lý, container sẽ **OOM kill lặp lại liên tục**, tạo thành restart loop.

**Cách xác nhận một container bị OOM kill (kỹ năng debug bắt buộc):**

```
docker inspect <container> --format '{{.State.OOMKilled}}'
```

Lệnh này trả về `true` nếu lần dừng gần nhất là do OOM Killer — đây là bằng chứng xác thực, không nên chỉ đoán qua exit code 137 (vì `docker kill` mặc định cũng dùng `SIGKILL` và cũng cho ra exit code 137, nhưng không phải do OOM).

### 3.4 `--memory-reservation` — ngưỡng mềm, khác `--memory`

Nhiều người mới học nhầm lẫn `--memory` và `--memory-reservation`. Phân biệt rõ:

- `--memory` (giới hạn cứng): Container **không bao giờ** được vượt quá giá trị này — vượt là bị OOM kill ngay.
- `--memory-reservation` (ngưỡng mềm/soft limit): Chỉ có tác dụng khi **toàn bộ host** đang chịu áp lực bộ nhớ. Trong điều kiện bình thường, container vẫn có thể dùng nhiều RAM hơn giá trị này (miễn không vượt `--memory` cứng nếu có đặt). Khi host thiếu RAM, kernel sẽ ưu tiên "ép" các container về dưới ngưỡng `--memory-reservation` của chúng trước khi phải OOM kill.

Thực tế production: đặt `--memory-reservation` thấp hơn `--memory` một khoảng hợp lý, dùng cho các container không quan trọng bằng, để nhường RAM cho các container ưu tiên cao hơn khi host căng thẳng tài nguyên — mà không cần OOM kill ngay lập tức.

## 4. Restart Policy — Self-Healing trong Production

### 4.1 Bốn chính sách

| Policy | Hành vi |
|---|---|
| `no` | Mặc định. Container **không bao giờ** tự khởi động lại, dù thoát với bất kỳ exit code nào hay Docker daemon restart. |
| `on-failure[:max-retries]` | Chỉ tự khởi động lại nếu container thoát với **exit code khác 0** (tức là thoát do lỗi). Có thể giới hạn số lần thử lại tối đa bằng `:max-retries` (ví dụ `on-failure:5`). Nếu container thoát "sạch" (exit code 0, tức chủ động dừng thành công), **không** tự khởi động lại. |
| `always` | Luôn tự khởi động lại **bất kể exit code nào** — kể cả khi bạn chủ động `docker stop` container, nó vẫn tự khởi động lại **trừ khi** Docker daemon cũng bị restart, hoặc bạn dừng bằng chính `docker stop` (Docker ghi nhớ trạng thái "người dùng chủ động dừng" để không tự khởi động lại ngay sau đó — nhưng nếu daemon restart, container `always` sẽ tự chạy lại). |
| `unless-stopped` | Giống `always`, nhưng nếu bạn **chủ động** `docker stop` container, nó sẽ **không** tự khởi động lại kể cả khi Docker daemon restart (khác biệt cốt lõi với `always`). |

### 4.2 Chọn policy nào cho workload nào — quyết định thực chiến

- **`no`**: Dùng cho job chạy một lần (batch job, script migrate database, tác vụ CI) — bạn **không muốn** nó tự chạy lại nếu lỡ thất bại, vì có thể gây tác dụng phụ khi chạy trùng lặp (ví dụ migrate database chạy 2 lần).
- **`on-failure:N`**: Dùng cho worker/consumer xử lý message queue — nếu crash do lỗi tạm thời (mất kết nối mạng, timeout), tự thử lại là hợp lý, nhưng giới hạn số lần thử để tránh crash loop vô hạn nếu lỗi là do bug thật sự trong code (không tự khỏi dù thử bao nhiêu lần).
- **`always`**: Dùng cho service hạ tầng lõi cần chạy liên tục 24/7 và bạn muốn nó **luôn** sống lại sau khi server reboot (kể cả khi bạn quên chưa cấu hình container tự khởi động cùng hệ thống) — ví dụ reverse proxy, service mesh sidecar.
- **`unless-stopped`**: Lựa chọn phổ biến nhất cho **web service production** thông thường (API backend, web app). Lý do: nếu bạn chủ động dừng để bảo trì (`docker stop`), bạn **không muốn** nó tự bật lại ngoài ý muốn khi server reboot giữa lúc đang bảo trì — nhưng vẫn muốn nó tự hồi phục sau crash bất ngờ hoặc sau khi server reboot bình thường (không phải do bạn chủ động dừng).

### 4.3 "Self-healing" nghĩa là gì, và giới hạn của nó

Restart policy tạo cảm giác container "tự lành" — crash thì tự đứng dậy chạy lại, không cần con người can thiệp lúc nửa đêm. Đây là giá trị vận hành rất lớn. Nhưng **restart policy không phải phép màu**:

- Nó **không sửa nguyên nhân gốc**. Nếu container crash vì bug code hoặc vì thiếu RAM, restart liên tục chỉ che giấu triệu chứng — container sẽ rơi vào **restart loop** (xem chi tiết ở [[06-Troubleshooting]]).
- Nó **không thay thế health check**. Restart policy chỉ phản ứng khi tiến trình chính **thoát hẳn** — nếu ứng dụng bị "treo" (process vẫn chạy nhưng không phản hồi request nào, deadlock) mà không thoát, restart policy **không kích hoạt**. Đây là lý do cần kết hợp với `HEALTHCHECK` trong Dockerfile hoặc orchestrator (Kubernetes liveness probe — sẽ học ở khóa 05) để phát hiện và xử lý cả trường hợp "treo nhưng không chết".

## 5. Logging — Cơ chế và Rủi ro Disk Đầy

### 5.1 Logging driver mặc định: `json-file`

Theo mặc định, Docker capture toàn bộ output từ `stdout`/`stderr` của tiến trình chính trong container, và ghi vào một file JSON trên host (thường nằm dưới `/var/lib/docker/containers/<container-id>/`). Đây là logging driver `json-file` — mỗi dòng log ứng dụng in ra được bọc thành một object JSON kèm timestamp, rồi ghi nối tiếp vào file.

### 5.2 Sự cố thực tế: log làm đầy disk

Đây là **một trong những sự cố production phổ biến nhất** liên quan tới Docker mà gần như mọi sysadmin đều gặp ít nhất một lần: theo mặc định, **file log json-file không có giới hạn kích thước**. Một ứng dụng ghi log liên tục (đặc biệt log ở mức `DEBUG` để lại trong production do quên đổi, hoặc một vòng lặp lỗi in ra log liên tục hàng nghìn dòng/giây) sẽ khiến file log phình to không giới hạn — cho tới khi ổ đĩa host báo `No space left on device`, kéo theo hàng loạt hệ quả: container khác không ghi được file, Docker daemon không tạo được container mới, thậm chí hệ điều hành host mất ổn định nếu phân vùng chứa log Docker trùng với phân vùng hệ thống.

### 5.3 Giải pháp: giới hạn bằng `max-size` và `max-file`

Docker hỗ trợ cấu hình log rotation ngay trong logging driver `json-file` qua 2 tham số:

- `max-size`: Kích thước tối đa của **một** file log trước khi bị rotate (ví dụ `10m` = 10 MB).
- `max-file`: Số lượng file log được giữ lại tối đa (file cũ nhất bị xóa khi vượt số lượng này).

Ví dụ: `max-size=10m`, `max-file=3` nghĩa là tổng dung lượng log tối đa cho container đó là khoảng 30MB (3 file, mỗi file tối đa 10MB) — dù ứng dụng có ghi log bao nhiêu, dung lượng chiếm dụng luôn có trần.

**Cấu hình này có thể đặt theo 2 cách:**

1. **Per-container**, bằng flag khi `docker run` (xem chi tiết lệnh ở [[04-Commands]]).
2. **Toàn cục cho daemon**, trong file `/etc/docker/daemon.json` — áp dụng mặc định cho **mọi container mới** tạo sau đó (không ảnh hưởng container đã tồn tại). Đây là cách khuyến nghị cho môi trường production: đặt giới hạn mặc định ở cấp daemon để không ai "quên" cấu hình khi tạo container mới.

**Bài học thực chiến:** Không bao giờ để logging driver mặc định chạy không giới hạn trong production. Đây là dạng sự cố "âm thầm tích lũy" — không gây vấn đề gì trong nhiều tuần đầu, rồi đột ngột bùng phát thành sự cố nghiêm trọng đúng lúc không ai để ý.

## 6. `docker exec` vs `docker attach` — khác biệt sống còn khi debug

Đây là công cụ chính để debug container **đang chạy** trong production mà không được phép dừng nó.

- **`docker attach`**: Kết nối **trực tiếp** vào `stdin`/`stdout`/`stderr` của **tiến trình chính (PID 1)** đang chạy trong container — giống như bạn đang "nhìn qua vai" đúng tiến trình đó. Rủi ro lớn: nếu bạn gõ nhầm hoặc nhấn `Ctrl+C`, tín hiệu có thể truyền tới tiến trình chính và **làm dừng luôn container** (tùy ứng dụng xử lý signal thế nào). Ít khi dùng để debug chủ động trong production.
- **`docker exec`**: Khởi chạy một **tiến trình hoàn toàn mới** (ví dụ `/bin/bash`, `/bin/sh`, hoặc bất kỳ lệnh nào có trong image) **bên trong cùng namespace** (network, PID, filesystem...) với container đang chạy — nhưng là một tiến trình con độc lập, không đụng chạm tới tiến trình chính. Đây là công cụ debug an toàn và phổ biến nhất: mở shell vào container đang chạy để kiểm tra file, biến môi trường, network, chạy lệnh chẩn đoán — **không ảnh hưởng** gì tới tiến trình chính đang phục vụ traffic thật.

**Quy tắc thực chiến:** Luôn dùng `docker exec` để debug. Chỉ dùng `docker attach` khi bạn **thật sự** cần theo dõi trực tiếp output/input của tiến trình chính (hiếm khi cần trong vận hành thường ngày), và luôn cẩn trọng khi thoát ra (dùng đúng tổ hợp phím detach `Ctrl+P, Ctrl+Q` thay vì `Ctrl+C` để không vô tình gửi signal cho tiến trình chính).

## 7. `docker inspect` và Go Template

`docker inspect` trả về toàn bộ metadata của container (hoặc image, network, volume) dưới dạng JSON khổng lồ — bao gồm trạng thái, cấu hình mạng, resource limit, mount, exit code, thời gian bắt đầu/kết thúc... Trong vận hành thật, bạn hiếm khi cần đọc toàn bộ JSON — bạn cần **trích xuất đúng một trường cụ thể**, đặc biệt khi viết script tự động hóa (đúng kỹ năng bạn đang học song song ở Ansible). Docker hỗ trợ `--format` với cú pháp **Go template** để làm việc này ngay trong CLI, không cần thêm `jq` hay xử lý JSON thủ công (dù `jq` vẫn là lựa chọn hợp lệ khác). Chi tiết cú pháp và ví dụ thực tế ở [[04-Commands]].

## 8. Bộ công cụ giám sát và dọn dẹp vận hành

Bốn công cụ này thường bị bỏ sót khi mới học Docker nhưng là **kỹ năng vận hành bắt buộc** hằng ngày:

- **`docker events`**: Stream các sự kiện xảy ra trên Docker daemon theo thời gian thực (container start/stop/die/oom, image pull, network connect...) — hữu ích để quan sát "chuyện gì đang xảy ra" khi debug sự cố phức tạp liên quan nhiều container, hoặc để tích hợp vào hệ thống giám sát/alerting của riêng bạn.
- **`docker stats`**: Hiển thị CPU%, memory usage/limit, network I/O, block I/O theo thời gian thực cho container đang chạy — công cụ đầu tiên bạn dùng khi nghi ngờ có container đang "ăn" quá nhiều tài nguyên.
- **`docker system df`**: Cho biết tổng dung lượng đĩa đang bị chiếm dụng bởi images, containers, local volumes, và **build cache** — thường là nguyên nhân bất ngờ khiến ổ đĩa đầy mà không ai để ý (build cache có thể chiếm hàng chục GB sau nhiều lần build image).
- **`docker system prune`**: Dọn dẹp tài nguyên không còn dùng (container đã exit, network không được container nào dùng, dangling image, build cache). Có các cờ mở rộng phạm vi xóa (`-a`, `--volumes`) — **cực kỳ rủi ro nếu dùng bừa trong production** vì có thể xóa nhầm image hoặc volume đang cần dùng lại sau này dù hiện tại không có container nào tham chiếu tới nó. Chi tiết rủi ro và cách dùng an toàn ở [[04-Commands]] và [[06-Troubleshooting]].

Tiếp theo: [[03-Architecture|03 — Kiến trúc]]
