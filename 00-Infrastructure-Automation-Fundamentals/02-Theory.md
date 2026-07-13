---
title: "02 - Theory"
module: 0
tags: [ansible, sysops-infra, module-00, theory, iac, idempotency, gitops]
---

# 02 — Lý thuyết

## 1. Traditional Infrastructure Management vs Infrastructure as Code

### Traditional (ClickOps / Manual)

Đây là cách bạn đã làm quen ở khóa Linux Sysadmin: SSH vào server, gõ lệnh, sửa file cấu hình bằng `vim`, restart service bằng tay. Cách này **đúng** cho mục đích học và cho môi trường 1-2 server, nhưng có 4 vấn đề chết người ở quy mô doanh nghiệp:

| Vấn đề | Hậu quả thực tế |
|---|---|
| **Không lặp lại được (Not Repeatable)** | Server A cấu hình lúc 2 giờ sáng thứ Hai, Server B cấu hình lúc 4 giờ chiều thứ Sáu bởi một kỹ sư khác — không ai đảm bảo hai máy giống nhau 100%. |
| **Configuration Drift** | Theo thời gian, người vận hành login sửa "tạm" một dòng config để fix gấp sự cố, quên ghi lại → hệ thống "trôi" dần khỏi trạng thái chuẩn ban đầu, không ai biết trạng thái thật là gì. |
| **Không có lịch sử thay đổi (No Audit Trail)** | Khi sự cố xảy ra, không ai trả lời được câu hỏi "ai đã đổi gì, lúc nào, tại sao" — vì thay đổi nằm trong đầu người kỹ sư, không nằm trong hệ thống nào cả. |
| **Không scale được** | Cấu hình 1 server mất 30 phút thủ công. Cấu hình 100 server theo cách này mất 50 giờ làm việc liên tục — không khả thi khi cần scale gấp (ví dụ Black Friday). |

Hậu quả gộp lại có một cái tên quen thuộc trong ngành: **"Snowflake Server"** — mỗi server là một "bông tuyết" độc nhất vô nhị, không cái nào giống cái nào, và không ai dám động vào vì không biết động vào sẽ hỏng cái gì.

### Infrastructure as Code (IaC)

IaC là cách quản lý và cấp phát hạ tầng (server, network, cấu hình) bằng **file code** (thường là YAML, HCL, JSON...) thay vì thao tác thủ công qua giao diện hoặc SSH tay. File code này được lưu trong Git, review như review code ứng dụng, và một công cụ (Ansible, Terraform...) đọc file đó rồi **áp dụng** lên hạ tầng thật.

So sánh trực tiếp:

| Tiêu chí | Traditional | IaC |
|---|---|---|
| Cách thay đổi hạ tầng | SSH + gõ lệnh tay | Sửa file code → chạy công cụ |
| Lặp lại (Repeatability) | Không đảm bảo | Đảm bảo — cùng file, cùng kết quả |
| Lịch sử thay đổi | Không có (trừ khi ghi log thủ công) | Có, nằm trong lịch sử Git commit |
| Review trước khi áp dụng | Không | Có (Pull Request/Merge Request) |
| Rollback | Khó, thường phải sửa tay ngược lại | Dễ — checkout lại version code cũ, apply lại |
| Tốc độ ở quy mô lớn | Chậm tuyến tính theo số server | Nhanh — một lệnh áp dụng cho hàng trăm server song song |
| Rủi ro lỗi con người | Cao (gõ nhầm lệnh, quên bước) | Thấp hơn (logic được test, review trước) |

**Lưu ý quan trọng:** IaC không loại bỏ hoàn toàn kỹ năng Linux Sysadmin bạn vừa học — nó **nhân bản** kỹ năng đó. Ansible module `user`, `systemd`, `firewalld` bên dưới vẫn gọi đúng những lệnh và file cấu hình bạn đã học (`/etc/passwd`, `systemctl`, `firewall-cmd`...). Nếu không hiểu Linux bên dưới, bạn sẽ không debug được khi Ansible báo lỗi.

## 2. Infrastructure Lifecycle

Hạ tầng không phải "làm một lần rồi xong" — nó có vòng đời lặp lại liên tục:

1. **Plan (Lập kế hoạch):** Xác định cần hạ tầng gì — bao nhiêu server, cấu hình gì, network nào.
2. **Write/Provision (Viết code / Cấp phát):** Viết file IaC mô tả trạng thái mong muốn (desired state), hoặc cấp phát tài nguyên (VM, cloud instance).
3. **Review:** Đồng nghiệp review đoạn code IaC qua Pull Request trước khi áp dụng lên môi trường thật — giống hệt quy trình review code ứng dụng.
4. **Apply/Configure:** Công cụ (Ansible) áp dụng cấu hình lên hạ tầng thật.
5. **Test/Validate:** Kiểm tra hạ tầng sau khi áp dụng có đúng như mong đợi không (service chạy chưa, port mở chưa).
6. **Monitor:** Theo dõi liên tục để phát hiện configuration drift hoặc sự cố.
7. **Update/Iterate:** Có yêu cầu mới → quay lại bước Plan, sửa code, review, apply lại.
8. **Decommission (Thu hồi):** Khi hạ tầng không còn cần, gỡ bỏ có kiểm soát — cũng qua code, không xóa tay.

Vòng đời này lặp đi lặp lại — đây là lý do IaC đặc biệt phù hợp với chu kỳ CI/CD: mỗi thay đổi hạ tầng đi qua đúng quy trình như một thay đổi code ứng dụng.

## 3. Declarative vs Imperative

Đây là một trong những khái niệm hay bị hỏi nhất khi phỏng vấn — và hay bị trả lời sai.

- **Imperative (Mệnh lệnh / Thủ tục):** Bạn mô tả **từng bước cụ thể** cần làm, theo đúng thứ tự. Giống viết một kịch bản bash: "chạy `useradd`, rồi chạy `mkdir`, rồi chạy `chown`". Công cụ **không tự kiểm tra** trạng thái hiện tại trước khi chạy — nó chỉ thực thi đúng những gì bạn viết, theo đúng thứ tự.
- **Declarative (Khai báo):** Bạn mô tả **trạng thái mong muốn cuối cùng** (desired state), không quan tâm các bước trung gian. Ví dụ: "user `deploy` phải tồn tại, có home directory, thuộc group `docker`". Công cụ tự so sánh trạng thái hiện tại với trạng thái mong muốn, và **tự quyết định** cần làm gì (hoặc không cần làm gì) để đạt được trạng thái đó.

**Ansible nằm ở đâu?** Ansible là công cụ **chủ yếu Declarative**, nhưng **playbook lại được thực thi theo thứ tự tuần tự từ trên xuống (giống Imperative)**. Đây chính là điểm gây nhầm lẫn nhất khi mới học:

- Mỗi **task** trong Ansible playbook khai báo trạng thái mong muốn (ví dụ module `ansible.builtin.package: name=nginx state=present` nghĩa là "nginx phải được cài", không phải "chạy lệnh `apt install nginx`").
- Nhưng **các task chạy tuần tự, theo đúng thứ tự bạn viết trong file** — task 2 luôn chạy sau khi task 1 xong. Đây là phần mang tính Imperative.

Nói cách khác: Ansible declarative **ở cấp độ từng task** (module tự quyết định "làm gì" để đạt trạng thái mong muốn), nhưng imperative **ở cấp độ playbook** (bạn kiểm soát rõ ràng thứ tự thực thi). Đây là lý do Ansible được xem là dễ học hơn Terraform (thuần declarative, không đảm bảo thứ tự) đối với người quen viết script — bạn vẫn đọc playbook từ trên xuống như đọc một kịch bản.

## 4. Agent-based vs Agentless

Công cụ automation quản lý hạ tầng theo hai mô hình kiến trúc:

- **Agent-based (Puppet, Chef, SaltStack minion):** Phải **cài một phần mềm agent** chạy thường trực trên mỗi managed node. Agent định kỳ (ví dụ mỗi 30 phút) tự "gọi về" (pull) server trung tâm để lấy cấu hình mới nhất và tự áp dụng.
- **Agentless (Ansible):** **Không cần cài gì thêm** trên managed node ngoài Python (đã có sẵn trên hầu hết distro Linux) và một SSH server đang chạy — thứ mà mọi server Linux production đều có sẵn theo mặc định. Control node (máy bạn chạy lệnh Ansible) kết nối tới managed node qua SSH, đẩy (push) module Python cần thiết, thực thi, rồi dọn dẹp — không để lại tiến trình thường trực nào.

Đánh đổi thực tế:

| Tiêu chí | Agent-based | Agentless (Ansible) |
|---|---|---|
| Cài đặt ban đầu | Phải cài + đăng ký agent trên từng node | Chỉ cần SSH + Python có sẵn |
| Tài nguyên tiêu tốn trên managed node | Có tiến trình agent chạy thường trực (tốn RAM/CPU) | Không tốn tài nguyên khi không chạy task |
| Độ trễ cập nhật | Phụ thuộc chu kỳ pull của agent (vài phút đến vài chục phút) | Tức thời — chạy lệnh là áp dụng ngay (push model) |
| Bảo mật | Cần mở thêm cổng/giao thức riêng cho agent giao tiếp | Tận dụng lại kênh SSH đã được hardening (khóa Linux Sysadmin Module 20-26) |
| Quản lý ở quy mô rất lớn (hàng chục nghìn node) | Có lợi thế (pull tự động, không cần control node "gõ cửa" từng máy) | Cần thêm kiến trúc (Ansible Automation Platform, execution strategy) để scale tốt |

Đây chính là lý do Ansible được ưa chuộng làm công cụ **đầu vào** cho đội Sysadmin/DevOps: bạn không phải thay đổi gì trên hạ tầng hiện có, chỉ cần SSH key đã cấu hình đúng (khóa Linux Sysadmin đã dạy) là bắt đầu tự động hóa được ngay.

## 5. Idempotency — nguyên lý sống còn

**Định nghĩa:** Một thao tác là **idempotent** (khả nghịch/bất biến khi lặp) nếu chạy nó **một lần** hay **nhiều lần liên tiếp** đều cho ra **cùng một kết quả cuối cùng**, và những lần chạy sau (khi trạng thái đã đúng) **không gây thay đổi hay tác dụng phụ nào thêm**.

Ví dụ dễ hiểu bằng lệnh Linux bạn đã học:

| Lệnh | Idempotent? | Giải thích |
|---|---|---|
| `mkdir -p /opt/app` | Có | Chạy 10 lần, thư mục vẫn chỉ tồn tại đúng 1 lần, không lỗi, không tạo trùng. |
| `mkdir /opt/app` (không có `-p`, không kiểm tra tồn tại) | Không | Chạy lần 2 sẽ báo lỗi "File exists" — hành vi khác lần đầu. |
| `echo "127.0.0.1 app.local" >> /etc/hosts` | Không | Mỗi lần chạy lại **thêm một dòng mới** — chạy 5 lần thành 5 dòng trùng lặp. |
| `useradd deploy` | Không | Lần 2 báo lỗi "user already exists". |
| Ansible module `ansible.builtin.user: name=deploy state=present` | Có | Module tự kiểm tra user đã tồn tại chưa trước khi hành động — có thì báo `ok` (không đổi gì), chưa có thì mới tạo. |

**Tại sao idempotency quan trọng đến mức là nguyên lý cốt lõi của toàn khóa?**

1. **An toàn khi chạy lại (re-run safety):** Playbook Ansible chạy hỏng giữa chừng do mất mạng — bạn chạy lại **toàn bộ playbook từ đầu** mà không sợ tạo trùng user, ghi trùng dòng cấu hình, hay cài lại gói đã cài.
2. **Dùng được cho cả provisioning lẫn compliance check:** Chạy playbook định kỳ (ví dụ mỗi đêm) để "ép" mọi server quay về đúng trạng thái chuẩn — server nào bị configuration drift sẽ tự động được sửa lại, server nào đã đúng thì Ansible báo cáo "không có gì thay đổi" (task màu xanh `ok`, không phải `changed`).
3. **Là cơ sở cho báo cáo "changed" đáng tin cậy:** Khi Ansible báo task này `changed`, bạn biết chắc **có điều gì đó thật sự thay đổi** trên server — không phải noise. Đây là cơ chế báo cáo audit cực kỳ giá trị trong doanh nghiệp.

> Toàn bộ module Ansible chuẩn (`ansible.builtin.*`) được thiết kế idempotent theo mặc định. Khi bạn viết task dùng module `command` hoặc `shell` (chạy lệnh raw), **bạn phải tự đảm bảo idempotency** bằng `creates`, `removes`, hoặc `when` — đây là lỗi phổ biến nhất của người mới học Ansible, sẽ nói kỹ ở Module 03.

## 6. Mutable vs Immutable Infrastructure

- **Mutable Infrastructure:** Server tồn tại lâu dài (có thể hàng năm), được **sửa đổi tại chỗ** (in-place) mỗi khi cần cập nhật — vá bản vá bảo mật, đổi version phần mềm, sửa cấu hình. Đây là mô hình truyền thống và cũng là mô hình Ansible vận hành phổ biến nhất: bạn SSH (qua Ansible) vào server đang chạy và sửa nó.
- **Immutable Infrastructure:** Server **không bao giờ bị sửa sau khi đã triển khai**. Khi cần thay đổi (cập nhật version, vá lỗi), bạn **build một image/server mới hoàn toàn** từ code, triển khai thay thế server cũ, rồi **tiêu hủy** server cũ — không bao giờ SSH vào sửa server đang chạy.

| Tiêu chí | Mutable | Immutable |
|---|---|---|
| Cách cập nhật | Sửa tại chỗ (in-place) | Thay thế toàn bộ (replace) |
| Configuration drift | Có nguy cơ (theo thời gian tích lũy thay đổi) | Gần như không thể (server luôn build lại từ code) |
| Rollback | Khó — phải "sửa ngược" | Dễ — chỉ cần triển khai lại image cũ |
| Công cụ tiêu biểu | Ansible (chạy lặp trên server sống) | Docker image, Packer, AMI, Kubernetes Pod |
| Downtime khi cập nhật | Thấp (sửa tại chỗ, không cần thay máy) nhưng rủi ro drift cao | Cần chiến lược rolling/blue-green để tránh downtime |

**Liên hệ với lộ trình học:** Ansible (khóa này) vận hành tốt nhất với mô hình **Mutable** — quản lý server VM đang sống. Khi bạn học Docker (Part II ngay sau khóa này) và Kubernetes (khóa 05), bạn sẽ thấy tư duy **Immutable** chiếm ưu thế — container không bao giờ bị "vá" khi đang chạy, mà bị thay thế bằng container mới từ image mới. Hiểu rõ hai mô hình này giúp bạn biết **khi nào dùng Ansible, khi nào dùng container** trong một kiến trúc doanh nghiệp thật — thường là dùng **cả hai**: Ansible để chuẩn hóa OS nền (kể cả node chạy Docker/Kubernetes), container để đóng gói và triển khai ứng dụng.

## 7. GitOps — giới thiệu nhập môn

**GitOps** là một cách vận hành hạ tầng/ứng dụng trong đó **Git repository là nguồn chân lý duy nhất (single source of truth)** cho trạng thái mong muốn của toàn bộ hệ thống. Nguyên tắc cốt lõi:

1. Toàn bộ trạng thái mong muốn của hạ tầng (và thường cả ứng dụng) được mô tả bằng code, lưu trong Git.
2. Mọi thay đổi hạ tầng đi qua Git — commit, Pull Request, review, merge — **không có thay đổi nào được áp dụng trực tiếp mà không qua Git** (không còn "SSH vào sửa tay rồi quên ghi lại" như mô hình Traditional ở mục 1).
3. Một hệ thống tự động (CI/CD pipeline, hoặc agent chạy trong cluster) phát hiện thay đổi trong Git rồi tự động áp dụng lên hạ tầng thật, đảm bảo trạng thái thật luôn hội tụ (converge) về đúng trạng thái khai báo trong Git.

Ở mức nhập môn này, bạn chỉ cần nhớ: **Ansible Playbook + Git repository** đã là bước đầu tiên của GitOps — code của bạn (playbook, inventory, role) nằm trong Git, mọi thay đổi hạ tầng bắt đầu từ một commit. Khóa Kubernetes (05) sau này sẽ đi sâu vào GitOps đúng nghĩa với công cụ chuyên dụng (ví dụ ArgoCD, Flux) tự động đồng bộ liên tục — vượt ngoài phạm vi khóa này.

## 8. Enterprise IaC Architecture — bức tranh tổng thể

Trong một tổ chức thật, IaC không phải chỉ là "một người chạy `ansible-playbook` từ laptop". Kiến trúc thường gồm:

- **Source of Truth:** Git repository (GitHub/GitLab/Bitbucket) chứa toàn bộ playbook, role, inventory.
- **Review layer:** Pull Request/Merge Request — bắt buộc ít nhất 1 người review trước khi merge vào nhánh chính.
- **CI/CD pipeline:** Tự động chạy `ansible-lint`, syntax check, thậm chí `--check` (dry-run) khi có Pull Request, trước khi cho phép merge.
- **Control Node (Execution environment):** Máy hoặc container chạy `ansible-playbook` thật — thường **không phải** laptop cá nhân trong môi trường production, mà là một server/CI runner được cấp quyền và log lại đầy đủ.
- **Secrets management:** Không hardcode password/API key trong code — dùng Ansible Vault (Module 05) hoặc hệ thống secret tập trung (Vault by HashiCorp, AWS Secrets Manager...).
- **Target Infrastructure:** Managed node thật — VM on-premise, cloud instance (AWS EC2, Azure VM, GCP Compute Engine).
- **Observability:** Log lại mọi lần chạy playbook (ai chạy, khi nào, kết quả gì) để phục vụ audit và troubleshooting.

Sơ đồ trực quan cho kiến trúc này nằm ở [[03-Architecture|03 - Architecture]]. Từ Module 02 trở đi, bạn sẽ học từng mảnh ghép cụ thể của bức tranh này: control node, inventory, playbook, vault, role — và Module 07 (cuối Part I) sẽ ráp toàn bộ lại thành một hệ thống doanh nghiệp hoàn chỉnh.
