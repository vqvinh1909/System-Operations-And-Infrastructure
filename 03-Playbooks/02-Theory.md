---
title: "02 - Theory"
module: 3
tags: [ansible, sysops-infra, module-03, tasks, loops, conditionals, handlers, blocks, tags]
---

# 02 — Lý thuyết

## 1. Cấu trúc một Playbook

Một Playbook là **danh sách các Play**. Mỗi Play áp dụng lên một nhóm host cụ thể, chứa danh sách Task:

```yaml
---
- name: Play 1 - Cau hinh web server
  hosts: webservers
  become: true
  vars:
    http_port: 8080
  tasks:
    - name: Task 1
      ansible.builtin.debug:
        msg: "Dang chay tren {{ inventory_hostname }}"

- name: Play 2 - Cau hinh database server
  hosts: dbservers
  become: true
  tasks:
    - name: Task 1
      ansible.builtin.debug:
        msg: "Play khac, host khac"
```

Mỗi Play có: `name` (mô tả, hiển thị trong log), `hosts` (pattern host/nhóm từ inventory), `become` (có nâng quyền sudo không), `vars` (biến cục bộ cho Play), và `tasks` (danh sách Task thực thi tuần tự). Nhiều Play trong một Playbook chạy **tuần tự theo thứ tự khai báo**, cho phép một Playbook điều phối nhiều loại host khác nhau trong cùng một lần chạy.

## 2. Task — đơn vị thực thi trong Playbook

Task gọi một Module (đã học ở Module 02) với tham số cụ thể. Mỗi Task **nên có `name`** mô tả rõ mục đích — không bắt buộc về mặt cú pháp nhưng là best practice bắt buộc trong thực tế: log chạy Playbook hiển thị theo `name`, thiếu `name` khiến log khó đọc và khó debug khi Playbook dài hàng chục Task.

```yaml
tasks:
  - name: Cai dat cac goi can thiet
    ansible.builtin.apt:
      name:
        - nginx
        - git
        - curl
      state: present
```

## 3. Variables trong Playbook

Ngoài `vars` khai báo trực tiếp trong Play, biến còn có thể tới từ: inventory (Module 02), `vars_files` (file YAML riêng), `register` (lưu kết quả một Task để dùng ở Task sau), và `--extra-vars` (dòng lệnh, ưu tiên cao nhất — chi tiết thứ tự ưu tiên đầy đủ học ở Module 05).

```yaml
tasks:
  - name: Kiem tra phien ban OS
    ansible.builtin.command: cat /etc/os-release
    register: os_info

  - name: Hien thi ket qua da luu
    ansible.builtin.debug:
      var: os_info.stdout
```

`register` là cơ chế quan trọng cho phép Task sau **dùng lại kết quả** Task trước — nền tảng cho logic điều kiện phức tạp (mục 5).

## 4. Loops — lặp một Task trên danh sách giá trị

Thay vì viết N Task giống hệt nhau cho N giá trị, dùng `loop`:

```yaml
tasks:
  - name: Cai nhieu goi cung luc
    ansible.builtin.apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - git
      - curl
```

`{{ item }}` là biến đặc biệt Ansible tự tạo, đại diện cho từng phần tử trong danh sách `loop`. Với danh sách phức tạp hơn (list chứa dictionary), có thể truy cập thuộc tính:

```yaml
tasks:
  - name: Tao nhieu user co thuoc tinh rieng
    ansible.builtin.user:
      name: "{{ item.name }}"
      groups: "{{ item.groups }}"
    loop:
      - { name: alice, groups: "sudo" }
      - { name: bob, groups: "docker" }
```

> Ghi chú lịch sử: cú pháp cũ `with_items` vẫn hoạt động (tương thích ngược) nhưng `loop` là cú pháp được khuyến nghị từ Ansible 2.5 trở đi — cú pháp mới nhất quán hơn, dùng chung cơ chế cho mọi kiểu lặp.

## 5. Conditionals — chạy Task có điều kiện

`when` cho phép Task chỉ chạy khi biểu thức đúng — cực kỳ hữu ích khi một Playbook phải chạy đúng trên nhiều hệ điều hành hoặc nhiều môi trường khác nhau:

```yaml
tasks:
  - name: Cai bang module apt (Debian/Ubuntu)
    ansible.builtin.apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"

  - name: Cai bang module dnf (RHEL/Rocky)
    ansible.builtin.dnf:
      name: nginx
      state: present
    when: ansible_os_family == "RedHat"
```

`when` thường kết hợp với Facts (`ansible_os_family`, đã học ở Module 02) hoặc với kết quả `register` từ Task trước:

```yaml
tasks:
  - name: Kiem tra file cau hinh co ton tai khong
    ansible.builtin.stat:
      path: /etc/myapp/config.yml
    register: config_check

  - name: Chi tao file mac dinh neu chua ton tai
    ansible.builtin.copy:
      src: default-config.yml
      dest: /etc/myapp/config.yml
    when: not config_check.stat.exists
```

## 6. Blocks — nhóm Task và xử lý lỗi có cấu trúc

`block` gom nhiều Task lại để: áp dụng chung một điều kiện/thuộc tính (`when`, `become`), hoặc xử lý lỗi theo mô hình try/except/finally quen thuộc:

```yaml
tasks:
  - name: Trien khai ung dung co xu ly loi
    block:
      - name: Dung service truoc khi cap nhat
        ansible.builtin.systemd:
          name: myapp
          state: stopped

      - name: Cap nhat ma nguon ung dung
        ansible.builtin.git:
          repo: "https://git.example.com/myapp.git"
          dest: /opt/myapp
    rescue:
      - name: Neu buoc tren that bai, rollback ve ban cu
        ansible.builtin.command: /opt/myapp/scripts/rollback.sh
    always:
      - name: Luon khoi dong lai service, du thanh cong hay that bai
        ansible.builtin.systemd:
          name: myapp
          state: started
```

Đọc logic: `block` chạy các Task chính. Nếu **bất kỳ Task nào trong `block` thất bại**, Ansible dừng ngay `block`, nhảy sang `rescue` (tương đương `except` trong Python). `always` luôn chạy sau cùng, bất kể `block` thành công hay `rescue` đã kích hoạt (tương đương `finally`). Đây là công cụ xử lý lỗi mạnh nhất trong Playbook, quan trọng bậc nhất cho Zero Downtime Deployment ở Module 07.

## 7. Error Handling — `ignore_errors` và `failed_when`

Hai công cụ đơn giản hơn `block/rescue`, dùng khi không cần logic phức tạp:

| Công cụ | Ý nghĩa | Khi nào dùng |
|---|---|---|
| `ignore_errors: true` | Bỏ qua lỗi của Task này, Playbook tiếp tục chạy Task sau | Task không quan trọng, lỗi ở đây không nên chặn toàn bộ Playbook |
| `failed_when: <dieu-kien>` | Tự định nghĩa lại "thế nào là thất bại", ghi đè logic mặc định của module | Module trả `rc=0` (thành công theo Ansible) nhưng thực chất kết quả không đạt yêu cầu nghiệp vụ |

```yaml
tasks:
  - name: Kiem tra dung luong dia, khong chan Playbook neu lenh loi nho
    ansible.builtin.command: df -h /
    ignore_errors: true

  - name: Coi la that bai neu output chua tu ERROR
    ansible.builtin.command: /opt/myapp/scripts/healthcheck.sh
    register: health
    failed_when: "'ERROR' in health.stdout"
```

**Cảnh báo thực tế:** lạm dụng `ignore_errors: true` là lỗi tư duy phổ biến của người mới — nó âm thầm che giấu lỗi thật, khiến Playbook "chạy xong không báo lỗi" nhưng hệ thống thực tế đã sai trạng thái. Chỉ dùng khi bạn **chắc chắn** hiểu rõ và chấp nhận hậu quả của lỗi đó.

## 8. Handlers — chỉ hành động khi có thay đổi thật sự

Handler là Task đặc biệt, **chỉ chạy khi được `notify` bởi một Task khác đã thực sự thay đổi trạng thái** (`changed: true`). Trường hợp kinh điển: chỉ restart `nginx` khi file cấu hình vừa được sửa, không restart nếu không có gì thay đổi.

```yaml
tasks:
  - name: Copy file cau hinh nginx
    ansible.builtin.copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx

handlers:
  - name: Restart nginx
    ansible.builtin.systemd:
      name: nginx
      state: restarted
```

**Đặc điểm quan trọng cần nhớ:**
- Handler mặc định chạy **ở cuối Play**, sau khi toàn bộ Task đã chạy xong — không chạy ngay lập tức tại vị trí `notify`.
- Nếu nhiều Task cùng `notify` một Handler, Handler đó **chỉ chạy một lần duy nhất** ở cuối, dù được gọi bao nhiêu lần.
- Nếu cần Handler chạy ngay giữa Playbook (trước khi tới cuối Play), dùng `meta: flush_handlers`.
- Tên Handler trong `notify` phải khớp chính xác với `name` khai báo trong `handlers`.

## 9. Tags — chạy chọn lọc một phần Playbook

Gắn `tags` cho Task để có thể chạy (hoặc bỏ qua) riêng phần đó mà không cần chạy toàn bộ Playbook:

```yaml
tasks:
  - name: Cai dat goi (cham, chi can chay 1 lan dau)
    ansible.builtin.apt:
      name: nginx
      state: present
    tags: [install]

  - name: Cap nhat file cau hinh (chay thuong xuyen)
    ansible.builtin.copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    tags: [config]
```

Chạy chọn lọc: `ansible-playbook site.yml --tags config` (chỉ chạy Task có tag `config`), hoặc `ansible-playbook site.yml --skip-tags install` (chạy mọi thứ trừ tag `install`). Hữu ích trong thực tế khi Playbook lớn có bước cài đặt chậm (chạy một lần) và bước cập nhật cấu hình nhanh (chạy thường xuyên).

## 10. Include vs Import — động vs tĩnh

Cả hai dùng để tách Playbook/Task lớn thành nhiều file nhỏ dễ quản lý, nhưng khác nhau về **thời điểm resolve**:

| Đặc điểm | `import_tasks` / `import_playbook` | `include_tasks` |
|---|---|---|
| Thời điểm xử lý | Lúc **parse** Playbook (tĩnh, trước khi chạy) | Lúc **chạy** tới dòng đó (động, runtime) |
| Dùng `when` ở cấp include/import | Áp dụng cho **từng Task bên trong** file được import | Áp dụng cho **toàn bộ khối include** như một Task duy nhất |
| Dùng biến để chọn file động (`{{ bien }}.yml`) | Không hỗ trợ tốt (phải biết trước lúc parse) | Hỗ trợ đầy đủ |
| Tags | Tag của Task gốc được giữ nguyên | Cần khai báo tag rõ ràng ở cấp include |
| Hiệu năng | Nhanh hơn (xử lý một lần lúc parse) | Chậm hơn một chút (xử lý mỗi lần chạy) |

```yaml
# Import - tinh, quyet dinh luc parse
tasks:
  - import_tasks: install_packages.yml

# Include - dong, quyet dinh luc chay, ho tro bien
tasks:
  - include_tasks: "{{ os_family }}_setup.yml"
```

**Nguyên tắc chọn:** dùng `import` khi cấu trúc Task cố định, không đổi theo điều kiện runtime — dùng `include` khi cần chọn file linh hoạt dựa trên biến hoặc facts (ví dụ chọn file cấu hình theo `ansible_os_family` chỉ biết được lúc chạy).
