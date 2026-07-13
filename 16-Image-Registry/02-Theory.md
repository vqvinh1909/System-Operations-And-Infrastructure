---
title: "02 - Theory"
module: 16
tags: [docker, sysops-infra, module-16, registry, harbor, oci]
---

# 02 — Lý thuyết: Registry, OCI Distribution Spec, Docker Hub vs Private Registry vs Harbor

## Registry là gì — định nghĩa chính xác

Một **container registry** là một dịch vụ lưu trữ và phân phối image theo chuẩn giao thức HTTP API gọi là **OCI Distribution Specification** (trước đây gọi là Docker Registry HTTP API V2 — Docker đã đóng góp chuẩn này cho Open Container Initiative để nó trở thành chuẩn mở, không phụ thuộc riêng Docker). Bất kỳ công cụ nào nói đúng giao thức này — `docker`, `podman`, `containerd`, `crane` — đều push/pull được vào bất kỳ registry nào tuân thủ chuẩn, kể cả khi registry đó không phải do Docker Inc. vận hành.

Điểm quan trọng cần hiểu: **registry không lưu "một file image"**. Nó lưu hai loại đối tượng tách biệt:

- **Blob**: từng layer riêng lẻ của image (dữ liệu nhị phân nén), và file cấu hình image (image config JSON).
- **Manifest**: một file JSON mô tả một image cụ thể được ghép từ những blob nào, theo thứ tự nào, chạy trên kiến trúc CPU/OS nào (`linux/amd64`, `linux/arm64`...).

Khi bạn `docker pull nginx:1.27`, thực chất client đang: (1) hỏi registry lấy manifest ứng với tag `1.27`, (2) đọc trong manifest thấy cần những blob (digest) nào, (3) kiểm tra local đã có blob nào rồi (theo digest, không theo tên) để **không tải lại layer đã có** — đây chính là lý do vì sao pull một image có layer trùng với image bạn đã có trước đó lại nhanh bất thường.

## Docker Hub

Docker Hub là registry công khai do Docker Inc. vận hành, mặc định mọi client Docker trỏ tới nếu không cấu hình gì khác. Vai trò đúng của nó trong doanh nghiệp: **nguồn base image chính thống** (`ubuntu`, `alpine`, `postgres`, `nginx`...) — những image bạn `FROM` trong Dockerfile, không phải nơi lưu image sản phẩm nội bộ.

Docker Hub có giới hạn **pull rate limit** cho tài khoản miễn phí/ẩn danh (thường gây sự cố CI/CD khi build chạy quá nhiều lần trong ngày và bị chặn) — đây là một lý do thực tế nữa khiến doanh nghiệp thường dựng một **pull-through cache** (registry nội bộ đóng vai trò cache, kéo và giữ lại image từ Docker Hub) thay vì để mọi server gọi thẳng ra Docker Hub.

## Vì sao doanh nghiệp KHÔNG dùng thẳng Docker Hub cho image nội bộ

Đây là câu hỏi cốt lõi của cả module. Bốn lý do thực tế:

1. **Bảo mật/rò rỉ dữ liệu**: Image nội bộ thường chứa (dù vô tình hay cố ý) thông tin nhạy cảm — cấu hình kết nối nội bộ, đôi khi cả secret bị bake nhầm vào layer. Nếu repo là public (hoặc bị lộ do quản lý quyền lỏng lẻo), toàn bộ layer lịch sử vẫn tải được kể cả sau khi bạn xóa file ở layer sau — vì layer cũ vẫn còn trong registry.
2. **Kiểm soát truy cập chi tiết (RBAC)**: Doanh nghiệp cần phân quyền theo team/project — team A chỉ push được vào project A, team B không pull được image của team C. Docker Hub free/team tier không đủ chi tiết cho mô hình tổ chức nhiều team.
3. **Tuân thủ và audit (compliance)**: Nhiều ngành (tài chính, y tế) yêu cầu chứng minh được image nào được ai push lúc nào, đã qua quét lỗ hổng bảo mật hay chưa, trước khi được phép chạy production — cần audit log nội bộ, không thể phụ thuộc dịch vụ bên thứ ba.
4. **Chi phí băng thông và độ trễ**: Hàng trăm server pull image nhiều lần mỗi ngày qua internet ra Docker Hub vừa tốn băng thông vừa chậm hơn nhiều so với pull từ registry nằm cùng datacenter/VPC.

## Private Registry tự host (`registry:2`)

Docker cung cấp sẵn image chính thức `registry:2` (dự án mã nguồn mở **Distribution**) — chạy một registry tuân thủ chuẩn OCI Distribution Spec chỉ bằng một lệnh `docker run`. Đây là cách nhanh nhất để có một registry nội bộ hoạt động được.

Tuy nhiên `registry:2` mặc định **chỉ là API lưu trữ thuần túy** — không có:
- Giao diện web để xem danh sách image/tag.
- Cơ chế xác thực người dùng theo project/team (RBAC) — mặc định không yêu cầu đăng nhập, hoặc nhiều nhất chỉ có Basic Auth dùng chung một cặp user/password cho tất cả.
- Image scanning tích hợp.
- Replication (đồng bộ image giữa nhiều site/datacenter).

Vì vậy `registry:2` phù hợp cho: lab học tập, môi trường CI nội bộ nhỏ, hoặc làm registry cache đơn giản — **không phù hợp làm registry chính cho một doanh nghiệp có nhiều team**.

## Harbor — registry doanh nghiệp

Harbor là một registry mã nguồn mở do VMware khởi xướng, hiện là dự án **CNCF Graduated** (đạt mức trưởng thành cao nhất của Cloud Native Computing Foundation từ tháng 6/2020 — cùng nhóm với Kubernetes, Prometheus). Harbor về bản chất **bọc thêm một lớp quản trị doanh nghiệp lên trên chính registry Distribution** (cùng công nghệ lõi với `registry:2`), cộng thêm:

- **Giao diện web (Portal)** quản lý project, user, image, tag trực quan.
- **RBAC theo project**: mỗi project có danh sách thành viên với vai trò riêng (Guest chỉ pull, Developer push/pull, Maintainer, Project Admin...).
- **Tích hợp image scanning** (mặc định dùng **Trivy** — công cụ quét lỗ hổng mã nguồn mở phổ biến) ngay trong luồng push, có thể chặn deploy nếu image chứa lỗ hổng mức nghiêm trọng theo policy.
- **Replication**: đồng bộ image giữa nhiều Harbor instance (ví dụ giữa datacenter chính và DR site, hoặc giữa on-prem và cloud).
- **Robot accounts**: tài khoản kỹ thuật dành riêng cho CI/CD pipeline push/pull tự động, tách biệt khỏi tài khoản người dùng thật — thực hành bảo mật quan trọng để không phải nhúng mật khẩu cá nhân vào pipeline.
- **Garbage Collection** có lịch, giúp registry không phình vô hạn theo thời gian.
- **Content trust / image signing** (dựa trên Notary/Cosign) để đảm bảo image chạy production đúng là image đã được build và ký bởi pipeline hợp lệ, không bị thay thế giữa chừng.

> [!note] Vì sao học `registry:2` trước khi học Harbor
> Harbor dùng chính engine Distribution (`registry:2`) làm thành phần lõi lưu trữ blob/manifest bên trong nó. Hiểu rõ `registry:2` hoạt động thế nào giúp bạn hiểu Harbor không phải "một sản phẩm hoàn toàn khác" mà là registry chuẩn + một hệ sinh thái quản trị bọc xung quanh — điều này rất hữu ích khi troubleshoot Harbor, vì nhiều lỗi thực chất là lỗi ở tầng Distribution bên dưới.

## Image Security — Image Scanning và Vulnerability Database

**Image scanning** là quá trình phân tích từng layer của image để liệt kê ra toàn bộ package/thư viện đã cài (gọi là **SBOM — Software Bill of Materials**), sau đó đối chiếu danh sách này với một **vulnerability database** (cơ sở dữ liệu lỗ hổng đã biết, ví dụ **NVD — National Vulnerability Database**, GitHub Advisory Database, và các advisory riêng của từng distro) để tìm ra package nào đang chứa lỗ hổng đã được công bố (mỗi lỗ hổng có mã định danh **CVE**).

Hai công cụ phổ biến nhất hiện tại (đã xác nhận qua WebSearch, tính đến 07/2026):
- **Trivy** (mã nguồn mở, của Aqua Security) — công cụ scan mặc định được tích hợp sẵn trong Harbor.
- **Docker Scout** — công cụ scan chính thức của Docker, đã thay thế lệnh `docker scan` cũ (vốn dựa trên engine của Snyk), tích hợp sẵn từ Docker Desktop 4.17 trở lên, tổng hợp dữ liệu lỗ hổng từ khoảng 23 nguồn advisory khác nhau.

Kết quả scan thường phân loại theo mức độ nghiêm trọng: **Critical, High, Medium, Low** (dựa trên điểm **CVSS — Common Vulnerability Scoring System**). Một chính sách bảo mật doanh nghiệp điển hình: "image có lỗ hổng mức Critical thì pipeline CI/CD tự động fail, không cho deploy" — đây là lúc image scanning chuyển từ "công cụ tham khảo" thành "gate bắt buộc" trong pipeline.

> [!warning] Scan không phải một lần duy nhất
> Một image "sạch" tại thời điểm build có thể trở nên "có lỗ hổng" vài tuần sau — không phải vì image thay đổi, mà vì vulnerability database cập nhật thêm CVE mới được phát hiện cho các package đã có sẵn trong image. Đây là lý do Harbor hỗ trợ lên lịch **re-scan định kỳ** cho toàn bộ image đang lưu, không chỉ scan một lần lúc push.
