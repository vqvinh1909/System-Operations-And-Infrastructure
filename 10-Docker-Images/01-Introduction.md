---
title: "01 - Introduction"
module: 10
tags: [docker, sysops-infra, module-10, introduction]
---

# 01 - Introduction: Docker Images

## Image là gì, và tại sao phải hiểu sâu chứ không chỉ "biết dùng"

Ở Module 09, bạn đã biết: container là một **process** chạy trên host, được cô lập bằng namespace và giới hạn tài nguyên bằng cgroups. Nhưng process đó cần một **root filesystem** để chạy — cần có `/bin/sh`, cần có thư viện, cần có binary ứng dụng của bạn. **Image chính là root filesystem đóng gói sẵn đó**, cộng với metadata mô tả cách chạy nó (entrypoint, biến môi trường, working directory...).

Nói cách khác:

```
IMAGE  = Root filesystem (đóng băng, read-only) + Metadata (cách chạy)
CONTAINER = Image + 1 layer ghi được (writable) + Process đang chạy
```

Nhiều người mới học Docker chỉ dừng ở mức "copy Dockerfile mẫu trên mạng, sửa vài dòng, build, chạy được là xong". Nhưng trong môi trường doanh nghiệp thực tế, đây là những tình huống bạn sẽ gặp và **phải hiểu sâu mới xử lý được**:

- CI/CD pipeline build image mất 8 phút vì cache không hit, trong khi đồng nghiệp build local chỉ mất 20 giây — bạn phải hiểu cơ chế cache theo layer để biết dòng nào trong Dockerfile đang phá cache.
- Image production nặng 1.4GB trong khi ứng dụng Go compiled binary chỉ 15MB — bạn phải hiểu multi-stage build để tách phần build-time (compiler, source code, cache) ra khỏi image chạy thật.
- Security team quét image, báo lỗi "container chạy bằng root" — bạn phải hiểu instruction `USER` và vì sao mặc định Docker chạy root nếu không khai báo.
- Ổ đĩa `/var/lib/docker` trên build server đầy ("no space left on device") — bạn phải hiểu image được lưu theo layer, và layer không được dọn sẽ tích tụ theo thời gian.

Tất cả các tình huống trên đều bắt nguồn từ việc hiểu (hoặc không hiểu) cách Docker **xây dựng image từ Dockerfile**. Đó là trọng tâm của module này.

## Image khác gì so với container (nhắc lại, mở rộng)

| Đặc điểm | Image | Container |
|---|---|---|
| Trạng thái | Bất biến (immutable), read-only | Có 1 writable layer trên cùng |
| Vòng đời | Được build 1 lần, dùng lại nhiều lần | Được tạo, chạy, dừng, xóa |
| Lưu trữ | Nhiều layer xếp chồng (layer cache) | Image layers + container layer |
| Ví dụ thực tế | `myapp:1.4.2` trên registry | `docker run myapp:1.4.2` -> process cụ thể đang chạy |

Một ví dụ dễ hình dung cho học viên đã quen Linux: image giống như một bản ISO cài sẵn hệ điều hành tối giản, còn container giống như máy ảo được khởi động từ ISO đó — bạn có thể tạo hàng chục container (máy ảo) từ cùng một image (ISO) mà không ảnh hưởng lẫn nhau.

## Dockerfile là gì

Dockerfile là một văn bản thuần text, chứa danh sách **instruction** (chỉ thị) tuần tự, mô tả từng bước để build ra image. Mỗi instruction, khi build, sẽ tạo ra **một layer**. BuildKit (build engine mặc định từ Docker Engine 23.0 trở đi) đọc Dockerfile, thực thi từng bước, và xếp chồng các layer lại thành image hoàn chỉnh.

Ví dụ tối giản:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
CMD ["node", "server.js"]
```

Chỉ 6 dòng, nhưng mỗi dòng đều có ý nghĩa kỹ thuật riêng, ảnh hưởng tới cache, kích thước image, và bảo mật — tất cả sẽ được mổ xẻ trong [[02-Theory]].

## Vì sao SysAdmin/DevOps cần thành thạo Dockerfile (không chỉ để dev dùng)

Có một hiểu lầm phổ biến: "Dockerfile là việc của dev, ops chỉ cần biết `docker run`." Thực tế trong doanh nghiệp:

- Bạn (SysAdmin/DevOps) thường là người **review** Dockerfile trước khi merge, để đảm bảo không chạy root, không leak secret, image không quá nặng.
- Bạn là người viết Dockerfile cho các **infrastructure tool** nội bộ (ví dụ: đóng gói script backup, monitoring agent, Ansible control node vào container để chạy trong CI).
- Bạn là người chịu trách nhiệm khi build server hết dung lượng, khi image scan ra lỗ hổng bảo mật, khi pipeline build chậm bất thường — tất cả đều cần hiểu Dockerfile và cơ chế build.

## Những gì bạn sẽ học trong module này

1. [[02-Theory]] — toàn bộ instruction Dockerfile, cơ chế layer/cache, Internal Working của BuildKit.
2. [[03-Architecture]] — sơ đồ trực quan cách layer xếp chồng và multi-stage build hoạt động.
3. [[04-Commands]] — lệnh `docker build` thực tế, Dockerfile mẫu đầy đủ (kể cả multi-stage, cache mount).
4. [[05-Labs]] — thực hành từ cơ bản đến mini project.
5. [[06-Troubleshooting]] — xử lý lỗi build thực chiến.
6. [[07-Interview]] — câu hỏi phỏng vấn thường gặp về chủ đề này.

Tiếp theo, đọc [[02-Theory]] để hiểu sâu từng instruction và cơ chế bên dưới.
