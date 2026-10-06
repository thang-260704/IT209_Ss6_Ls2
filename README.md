# Bài 2: Cấu hình phân quyền Nhóm và sudoers bằng visudo

## Mục tiêu
- Tạo người dùng (user) và nhóm (group) mới trên Linux.
- Hiểu cơ chế `sudo` và file cấu hình `/etc/sudoers`.
- Cấp quyền **giới hạn** cho một nhóm: chỉ được chạy `systemctl start/stop/restart/status`, **không** được cấp toàn quyền root.

## Môi trường

| Thông tin | Giá trị |
|---|---|
| Máy chủ | VPS Ubuntu 22.04.5 LTS – `160.187.229.88` |
| User quản trị thao tác | `devops` (nhóm `sudo`) |
| Nhóm mới | `devops-admin` |
| User mới | `deployer` (chỉ thuộc `devops-admin`, **không** thuộc nhóm `sudo`) |

---

## Các bước thực hiện

### Bước 1 – Tạo nhóm và người dùng

```bash
sudo groupadd devops-admin
sudo adduser deployer
sudo usermod -aG devops-admin deployer
id deployer
```

```text
uid=1001(deployer) gid=1002(deployer) groups=1002(deployer),1001(devops-admin)
```

- `groupadd` tạo nhóm mới.
- `adduser` tạo user, thư mục home `/home/deployer` và đặt mật khẩu.
- `usermod -aG` **thêm** (append) user vào nhóm phụ mà không xoá các nhóm sẵn có.

### Bước 2 – Cấu hình sudoers bằng `visudo`

```bash
sudo visudo -f /etc/sudoers.d/devops-admin
```

Thêm dòng sau:

```sudoers
%devops-admin ALL=(ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl status *
```

Kiểm tra cú pháp toàn bộ cấu hình sudo:

```bash
sudo visudo -c
sudo cat /etc/sudoers.d/devops-admin
```

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/devops-admin: parsed OK
```

![Tạo user, group và cấu hình sudoers](images/01-setup.png)

#### Giải thích dòng cấu hình

```text
%devops-admin   ALL   =(ALL)   NOPASSWD:   /usr/bin/systemctl restart *, ...
│               │     │        │           └─ Danh sách lệnh được phép (đường dẫn tuyệt đối)
│               │     │        └─ Không yêu cầu nhập mật khẩu
│               │     └─ Được chạy lệnh dưới danh nghĩa user nào (ALL = mọi user, kể cả root)
│               └─ Áp dụng trên host nào (ALL = mọi máy)
└─ Dấu % nghĩa là NHÓM (không có % là một user)
```

#### Vì sao dùng `visudo` và file trong `/etc/sudoers.d/`?

- `visudo` **kiểm tra cú pháp trước khi lưu**. Nếu sửa trực tiếp `/etc/sudoers` bằng `nano` và gõ sai, `sudo` sẽ ngừng hoạt động hoàn toàn, và có thể mất luôn quyền quản trị máy chủ.
- File `/etc/sudoers` mặc định của Ubuntu có dòng `@includedir /etc/sudoers.d`. Mọi file trong thư mục này đều được nạp như một phần của `/etc/sudoers`.
- Tách quy tắc ra một file riêng giúp dễ quản lý: xoá file là thu hồi quyền, và không bị ghi đè khi gói `sudo` được cập nhật.
- `visudo -f` tự đặt quyền file là `0440` (`-r--r-----`), đúng yêu cầu bảo mật của sudo.

### Bước 3 – Kiểm tra bằng tài khoản `deployer`

```bash
sudo -iu deployer
whoami
sudo -l
sudo systemctl restart cron
sudo systemctl status cron --no-pager | head -n 3
sudo -n whoami
```

`sudo -iu deployer` mở một login shell của `deployer`, tương đương `su - deployer`. Điểm khác là lệnh này được thực hiện từ tài khoản quản trị `devops`, nên không cần nhập mật khẩu của `deployer`.

Log thực tế:

```text
deployer@wean2607:~$ whoami
deployer
deployer@wean2607:~$ sudo -l
Matching Defaults entries for deployer on wean2607:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User deployer may run the following commands on wean2607:
    (ALL) NOPASSWD: /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *,
        /usr/bin/systemctl status *
deployer@wean2607:~$ sudo systemctl restart cron
deployer@wean2607:~$ sudo systemctl status cron --no-pager | head -n 3
● cron.service - Regular background program processing daemon
     Loaded: loaded (/lib/systemd/system/cron.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-10-06 22:02:05 +07; 2s ago
deployer@wean2607:~$ sudo -n whoami
sudo: a password is required
```

![sudo -l và restart cron bằng deployer](images/02-deployer-test.png)

---

## Phân tích kết quả

| Kiểm tra | Kết quả | Ý nghĩa |
|---|---|---|
| `sudo -l` | Hiện quy tắc `(ALL) NOPASSWD: /usr/bin/systemctl ...` | Quy tắc của nhóm `%devops-admin` đã áp dụng cho `deployer` |
| `sudo systemctl restart cron` | Chạy ngay, **không hỏi mật khẩu** | Đúng yêu cầu `NOPASSWD` |
| `systemctl status cron` | `active (running) since ... 2s ago` | Dịch vụ vừa được khởi động lại thành công |
| `sudo -n whoami` | **Bị từ chối**: `a password is required` | Lệnh nằm ngoài danh sách được phép. `deployer` không có toàn quyền root, chỉ chạy được đúng các lệnh `systemctl` đã khai báo |

Đây là nguyên tắc **Least Privilege** (đặc quyền tối thiểu): tài khoản tự động hoá chỉ được làm đúng việc cần làm. Nếu tài khoản bị lộ, kẻ tấn công cũng không chiếm được toàn quyền root.

## Lưu ý bảo mật

Ký tự đại diện `*` cho phép restart **mọi** dịch vụ, kể cả `ssh`. Trong môi trường thực tế, nên giới hạn chặt hơn theo đúng dịch vụ cần quản lý, ví dụ:

```sudoers
%devops-admin ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl restart cron, /usr/bin/systemctl status *
```

## Đối chiếu yêu cầu

| Yêu cầu | Kết quả |
|---|---|
| Tạo nhóm `devops-admin` | ✅ |
| Tạo user `deployer` và đưa vào nhóm `devops-admin` | ✅ |
| Cấu hình sudoers bằng `visudo` cho nhóm `%devops-admin` | ✅ |
| Được chạy `systemctl start/stop/restart/status` với `NOPASSWD` | ✅ |
| `sudo -l` hiển thị đúng quy tắc | ✅ |
| `sudo systemctl restart cron` không hỏi mật khẩu | ✅ |
| Không cấp toàn quyền root cho `deployer` | ✅ (`sudo -n whoami` bị từ chối) |
