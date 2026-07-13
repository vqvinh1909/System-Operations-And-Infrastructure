---
title: "06 - Troubleshooting"
module: 0
tags: [ansible, sysops-infra, module-00, troubleshooting]
---

# 06 — Troubleshooting thường gặp

Module này chưa có "lỗi kỹ thuật" theo nghĩa lỗi command — nhưng có những **lỗi tư duy** cực kỳ phổ biến khi một Sysadmin/DevOps mới chuyển từ quản trị thủ công sang IaC. Nhận diện sớm những lỗi này quan trọng không kém lỗi cú pháp.

## Sự cố 1: Configuration Drift — "Tại sao playbook chạy `ok` mà server vẫn sai?"

**Triệu chứng:** Một server production bị một kỹ sư khác SSH vào sửa tay để "fix gấp" sự cố lúc nửa đêm — không cập nhật lại code IaC. Vài tuần sau, chạy lại playbook không thấy báo lỗi gì (`ok`), nhưng bạn phát hiện server này khác các server còn lại.

**Nguyên nhân gốc:** Ansible chỉ đảm bảo trạng thái **những gì được khai báo trong playbook** — nếu playbook không khai báo một cấu hình cụ thể (ví dụ một dòng trong file cấu hình tùy chỉnh), Ansible sẽ **không biết** và **không sửa** nó. Đây không phải lỗi của Ansible — đây là hậu quả của việc **không tuân thủ quy trình GitOps** (SSH sửa tay thay vì đi qua Git).

**Cách xử lý:**
- Thiết lập quy tắc tổ chức: **mọi thay đổi hạ tầng phải đi qua Git**, kể cả fix gấp (fix gấp thì làm hotfix branch, merge nhanh, vẫn qua review tối thiểu).
- Chạy playbook định kỳ (cron/CI schedule) ở chế độ `--check` để phát hiện drift sớm, không đợi audit mới phát hiện.
- Nếu phát hiện drift, **không sửa tay lại** — sửa code IaC cho khớp với thực tế mong muốn, rồi để Ansible tự đồng bộ.

## Sự cố 2: Nhầm lẫn Declarative với "không cần hiểu chi tiết"

**Triệu chứng:** Người mới học nghĩ rằng vì Ansible là "declarative", họ chỉ cần khai báo trạng thái mong muốn mà không cần hiểu Ansible **sẽ làm gì** bên dưới — dẫn đến viết playbook mà không lường được tác dụng phụ (ví dụ dùng module `service: state=restarted` trong một task chạy trên mọi server cùng lúc, gây restart đồng loạt và downtime toàn hệ thống).

**Nguyên nhân gốc:** Declarative mô tả **kết quả mong muốn**, không có nghĩa là **không quan trọng cách đạt được nó**. Task vẫn chạy tuần tự (phần Imperative của Ansible đã học ở [[02-Theory|02 - Theory]] mục 3), và mỗi module vẫn có tác dụng phụ thật trên hệ thống thật.

**Cách xử lý:** Luôn tự hỏi "task này khi chạy trên **managed node đang production** sẽ gây tác động gì" trước khi viết — sẽ học kỹ chiến lược rolling update, `serial`, `max_fail_percentage` ở Module 07.

## Sự cố 3: Coi nhẹ bước Review vì "chỉ là một dòng YAML nhỏ"

**Triệu chứng:** Một kỹ sư tự merge thẳng playbook sửa cấu hình firewall vào `main` mà không qua Pull Request vì nghĩ "chỉ đổi 1 dòng, không đáng review" — dòng đó vô tình mở nhầm port ra Internet, gây lỗ hổng bảo mật.

**Nguyên nhân gốc:** Trong IaC, "1 dòng YAML" có thể tác động tới hàng trăm server cùng lúc — mức độ ảnh hưởng khác hoàn toàn 1 dòng code ứng dụng chạy trên 1 service. Đây chính là lý do bước Review trong Infrastructure Lifecycle ([[03-Architecture|03 - Architecture]] Sơ đồ 1) không được bỏ qua dù thay đổi nhỏ tới đâu.

**Cách xử lý:** Thiết lập branch protection rule bắt buộc ít nhất 1 approval trước khi merge vào nhánh chính chạy production — không có ngoại lệ "thay đổi nhỏ".

## Sự cố 4: Hiểu sai Agentless nghĩa là "không cần chuẩn bị gì"

**Triệu chứng:** Kỹ sư mới nghĩ Ansible agentless nghĩa là "cắm là chạy" — thử chạy Ansible nhắm vào managed node chưa từng cấu hình SSH key, không có Python, hoặc user không có quyền `sudo` — rồi báo "Ansible bị lỗi".

**Nguyên nhân gốc:** Agentless nghĩa là không cần cài **thêm phần mềm agent chuyên dụng**, nhưng vẫn cần đúng 3 điều kiện tối thiểu: SSH server chạy, SSH key/auth hoạt động, Python có sẵn (hoặc cấu hình `ansible_python_interpreter`). Đây là kiến thức nền Linux Sysadmin (SSH, sudo) — không phải "phép màu" của Ansible.

**Cách xử lý:** Trước khi động vào Ansible, luôn tự kiểm tra lại 3 lệnh ở [[04-Commands|04 - Commands]] mục 1 — thói quen này sẽ tiết kiệm rất nhiều thời gian debug ở Module 02 khi mới bắt đầu học `ansible all -m ping`.
