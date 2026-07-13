---
title: "02 - Theory"
module: 20
tags: [docker, ansible, sysops-infra, module-20, capstone, haproxy, backup]
---

# 02 — Ôn tập lý thuyết nền cho kiến trúc 5 VM

Module này không giới thiệu khái niệm mới — phần này chỉ **kết nối lại** những gì bạn đã học rải rác, giải thích vì sao kiến trúc 5 VM được thiết kế đúng như vậy, để bạn hiểu **lý do** đằng sau mỗi quyết định trước khi bắt tay triển khai ở [[05-Labs]].

## 1. Vì sao tách riêng 5 VM thay vì chạy mọi thứ trên 1 máy

Một hạ tầng chạy mọi service trên một máy duy nhất có ba vấn đề: (1) một điểm lỗi duy nhất (single point of failure) — máy đó chết là toàn bộ hệ thống chết theo; (2) không mô phỏng đúng thực tế doanh nghiệp, nơi các thành phần thường tách vai trò rõ ràng vì lý do bảo mật (database không nên public trực tiếp) và vì lý do vận hành (scale riêng từng tầng theo nhu cầu thực tế — web cần scale ngang nhiều hơn database); (3) không luyện được kỹ năng Ansible thật sự (Ansible chỉ thể hiện giá trị rõ ràng khi quản lý nhiều node cùng lúc, không phải một node đơn lẻ).

Năm vai trò được chọn phản ánh đúng một kiến trúc web 3 tầng (3-tier) cơ bản nhất mà gần như mọi doanh nghiệp vừa và nhỏ đều triển khai dưới một hình thức nào đó:

| VM | Vai trò | Vì sao cần tách riêng |
|---|---|---|
| Control Node | Điều phối Ansible | Không phục vụ traffic thật, tách riêng để không lẫn với hạ tầng production |
| HAProxy | Load Balancer / entry point | Điểm duy nhất tiếp xúc trực tiếp traffic từ ngoài, cần tách để cô lập rủi ro |
| Web01, Web02 | Tầng ứng dụng | Hai node để có tính sẵn sàng cao (HA) — một node chết, node còn lại vẫn phục vụ |
| Database | Tầng dữ liệu | Tách riêng vì lý do bảo mật (không nên public) và vì đây là tài nguyên trạng thái (stateful) cần chính sách backup riêng biệt hoàn toàn khác tầng web (stateless) |

## 2. HAProxy — vì sao cần một Load Balancer thay vì trỏ DNS thẳng vào 2 web server

Nếu không có HAProxy, cách "đơn giản" để có 2 web server là dùng DNS round-robin (khai báo 2 bản ghi A cùng tên miền). Cách này có nhược điểm nghiêm trọng: DNS **không biết** một trong hai server đã chết — client vẫn tiếp tục nhận được IP của server đã chết theo TTL cache, gây lỗi cho một phần người dùng cho tới khi cache DNS hết hạn.

HAProxy giải quyết đúng vấn đề đó bằng **health check chủ động**: liên tục gửi request kiểm tra (thường là HTTP GET tới một endpoint cụ thể) tới từng backend theo chu kỳ, tự động loại một backend ra khỏi vòng xoay ngay khi nó không phản hồi đúng, và tự động đưa lại vào vòng xoay khi nó khỏe trở lại — toàn bộ diễn ra trong vài giây, không phụ thuộc TTL cache DNS nào cả.

Thuật toán phân phối traffic phổ biến của HAProxy: **round-robin** (lần lượt, đơn giản, phù hợp khi các backend tương đương nhau) và **leastconn** (ưu tiên backend đang có ít kết nối đang xử lý nhất, phù hợp khi request có thời gian xử lý không đồng đều).

## 3. Vì sao Web01 và Web02 phải "stateless" (không giữ trạng thái riêng)

Với một Load Balancer phân phối request luân phiên giữa hai node, một request của cùng một người dùng có thể rơi vào Web01 lần này, Web02 lần sau — nếu ứng dụng lưu trạng thái phiên (session) riêng trên từng node (ví dụ lưu session trong RAM cục bộ của mỗi container), người dùng sẽ bị đăng xuất ngẫu nhiên hoặc mất dữ liệu giỏ hàng giữa chừng tùy request rơi vào node nào.

Đây chính là lý do kiến trúc có **Redis** — dùng làm nơi lưu session/cache **tập trung**, cả Web01 và Web02 cùng đọc/ghi vào một Redis duy nhất (chạy trên VM Database), khiến cả hai node web thực sự trở thành "vô trạng thái" (stateless) đối với chính chúng, dù ứng dụng vẫn có trạng thái ở tầng dữ liệu tập trung.

## 4. Vì sao có RabbitMQ trong kiến trúc

Không phải mọi tác vụ nên xử lý ngay trong lúc trả response cho người dùng — ví dụ gửi email xác nhận, xử lý file upload nặng, đồng bộ dữ liệu sang hệ thống khác. Đưa các tác vụ này vào một **message queue** (RabbitMQ) cho phép web server chỉ cần "đẩy một message vào hàng đợi rồi trả response ngay lập tức", còn việc xử lý thật sự diễn ra bất đồng bộ ở một tiến trình khác (worker) đọc từ hàng đợi đó — giúp thời gian phản hồi cho người dùng nhanh hơn nhiều, đồng thời tách rời (decouple) sự cố của tác vụ nền khỏi trải nghiệm người dùng chính.

## 5. Backup — RPO/RTO và vì sao rsync khác Volume Backup

Hai khái niệm cốt lõi khi thiết kế chiến lược backup:

- **RPO (Recovery Point Objective)**: chấp nhận mất tối đa bao nhiêu dữ liệu (tính theo thời gian) nếu sự cố xảy ra ngay trước lần backup tiếp theo. Backup mỗi 24h nghĩa là RPO tối đa 24h.
- **RTO (Recovery Time Objective)**: chấp nhận mất tối đa bao lâu để khôi phục lại dịch vụ sau sự cố.

Hai công cụ backup trong dự án phục vụ hai mục đích khác nhau, không thay thế nhau:

- **rsync**: đồng bộ file/thư mục (mã nguồn, file cấu hình, file tĩnh) sang một nơi lưu trữ khác, hỗ trợ đồng bộ tăng dần (incremental — chỉ truyền phần thay đổi, nhanh hơn copy toàn bộ mỗi lần) — phù hợp cho dữ liệu dạng file.
- **Volume backup** (ví dụ `docker run --rm -v <volume>:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz /data`): đóng gói toàn bộ nội dung một Docker volume (thường chứa dữ liệu database) thành một file nén tại một thời điểm — phù hợp cho dữ liệu dạng database, nơi tính toàn vẹn (consistency) của toàn bộ tập dữ liệu tại đúng một thời điểm quan trọng hơn việc chỉ đồng bộ từng file thay đổi riêng lẻ.

## 6. Monitoring cho dự án này khác gì Module 17 đã học

Module 17 dạy bạn dựng **một** stack Prometheus/Grafana cho một host. Trong dự án này, bạn cần quyết định thêm: Prometheus/cAdvisor chạy tập trung ở đâu (thường đặt trên Control Node hoặc một VM riêng, để không cạnh tranh tài nguyên với chính các service đang được giám sát), và cấu hình scrape làm sao để một Prometheus duy nhất theo dõi được cAdvisor trên **cả 4 node còn lại** (Web01, Web02, HAProxy, Database) — đây là bài toán "giám sát tập trung nhiều host" mà Module 17 chưa đặt ra (vì module đó chỉ dựng stack trên một host duy nhất).
