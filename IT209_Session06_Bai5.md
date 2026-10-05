# Báo Cáo Thiết Kế Và Cấu Hình Dịch Vụ Systemd Cho Ứng Dụng Java

## 1. Nội dung tệp tin cấu hình dịch vụ Systemd (`java-app.service`)

Đường dẫn tệp tin trên hệ thống: `/etc/systemd/system/java-app.service`

```ini
[Unit]
Description=Java Spring Boot Demo Application
After=network.target

[Service]
Type=simple
User=java-runner
ExecStart=/bin/sh -c "echo Application started under user $(whoami); while true; do sleep 5; done"
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Phân tích chi tiết các chỉ thị cấu hình
- **Khối `[Unit]`:**
  - `Description`: Mô tả tóm tắt vai trò của dịch vụ trong hệ thống.
  - `After=network.target`: Đảm bảo dịch vụ chỉ được khởi động sau khi toàn bộ hạ tầng mạng cơ bản đã sẵn sàng hoạt động.
- **Khối `[Service]`:**
  - `Type=simple`: Loại dịch vụ tiêu chuẩn, tiến trình được gọi bởi `ExecStart` chính là tiến trình xử lý chính (Main Process).
  - `User=java-runner`: Chỉ định dịch vụ thực thi dưới quyền của tài khoản hệ thống `java-runner` (non-root), hạn chế tối đa rủi ro an ninh trong trường hợp ứng dụng bị tấn công khai thác.
  - `ExecStart`: Lệnh thực thi để khởi động ứng dụng.
  - `Restart=on-failure`: Chính sách tự động khởi động lại dịch vụ khi tiến trình kết thúc bất thường (mã thoát khác 0, bị kết thúc bởi tín hiệu ngoại trừ clean stop, hoặc timeout).
  - `RestartSec=10`: Khoảng thời gian chờ chính xác 10 giây trước khi Systemd thực thi lệnh khởi động lại tiến trình sau sự cố.
- **Khối `[Install]`:**
  - `WantedBy=multi-user.target`: Liên kết dịch vụ vào runlevel đa người dùng không giao diện đồ họa, giúp dịch vụ tự động kích hoạt cùng hệ điều hành khi máy chủ khởi động (Boot).

---

## 2. Quy trình thực hiện chi tiết và Nhật ký thực thi (Terminal Logs)

### Bước 1: Tạo tài khoản hệ thống `java-runner` (Không cấp quyền Shell login)
Sử dụng tham số `-r` (system account) và `-s /usr/sbin/nologin` để chặn hoàn toàn khả năng đăng nhập shell trực tiếp:
```bash
plinhle26@devops-vm1:~$ sudo useradd -r -s /usr/sbin/nologin java-runner
plinhle26@devops-vm1:~$ getent passwd java-runner
java-runner:x:999:988::/home/java-runner:/usr/sbin/nologin
```

---

### Bước 2: Soạn thảo và lưu tệp cấu hình dịch vụ Systemd
Tạo tệp tin unit service tại `/etc/systemd/system/java-app.service`:
```bash
plinhle26@devops-vm1:~$ sudo bash -c 'cat << "EOF" > /etc/systemd/system/java-app.service
[Unit]
Description=Java Spring Boot Demo Application
After=network.target

[Service]
Type=simple
User=java-runner
ExecStart=/bin/sh -c "echo Application started under user $(whoami); while true; do sleep 5; done"
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF'
plinhle26@devops-vm1:~$ cat /etc/systemd/system/java-app.service
[Unit]
Description=Java Spring Boot Demo Application
After=network.target

[Service]
Type=simple
User=java-runner
ExecStart=/bin/sh -c "echo Application started under user $(whoami); while true; do sleep 5; done"
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

### Bước 3: Nạp lại cấu hình Systemd và kích hoạt khởi động cùng hệ thống
Thực thi `daemon-reload` để Systemd nhận diện unit file mới và tạo symbolic link tự động khởi động:
```bash
plinhle26@devops-vm1:~$ sudo systemctl daemon-reload
plinhle26@devops-vm1:~$ sudo systemctl enable java-app.service
Created symlink '/etc/systemd/system/multi-user.target.wants/java-app.service' → '/etc/systemd/system/java-app.service'.
```

---

### Bước 4: Khởi chạy dịch vụ và kiểm tra trạng thái hoạt động (`systemctl status`)
Khởi động dịch vụ và kiểm tra trạng thái tiến trình thực tế:
```bash
plinhle26@devops-vm1:~$ sudo systemctl start java-app.service
plinhle26@devops-vm1:~$ sudo systemctl status java-app.service
● java-app.service - Java Spring Boot Demo Application
     Loaded: loaded (/etc/systemd/system/java-app.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-10-05 13:26:47 UTC; 5s ago
 Invocation: 5b7b856c78bb485cb4e9cadd8674c3e8
   Main PID: 7243 (sh)
      Tasks: 2 (limit: 1060)
     Memory: 456K (peak: 1.8M)
        CPU: 54ms
     CGroup: /system.slice/java-app.service
             ├─7243 /bin/sh -c "echo Application started under user \$(whoami); while true; do sleep 5; done"
             └─7250 sleep 5

Oct 05 13:26:47 devops-vm1 systemd[1]: Started java-app.service - Java Spring Boot Demo Application.
Oct 05 13:26:47 devops-vm1 sh[7243]: Application started under user java-runner
```

**Nhận xét:**
- Trạng thái dịch vụ hiển thị `active (running)`.
- Log khởi tạo xác nhận ứng dụng chạy chính xác dưới danh nghĩa người dùng: `Application started under user java-runner`.

---

### Bước 5: Kiểm thử cơ chế tự phục hồi sự cố (Auto-Restart On Failure)
Giả lập sự cố ứng dụng bị sập đột ngột bằng cách gửi tín hiệu cưỡng chế dừng (`SIGKILL - 9`) tới Main PID `7243`:
```bash
plinhle26@devops-vm1:~$ systemctl show --property MainPID java-app.service
MainPID=7243
plinhle26@devops-vm1:~$ sudo kill -9 $(systemctl show --property MainPID --value java-app.service)
```

Theo dõi toàn bộ chu trình xử lý sự cố của Systemd thông qua `journalctl -u java-app.service -f`:
```bash
plinhle26@devops-vm1:~$ journalctl -u java-app.service -f
Oct 05 13:26:47 devops-vm1 systemd[1]: Started java-app.service - Java Spring Boot Demo Application.
Oct 05 13:26:47 devops-vm1 sh[7243]: Application started under user java-runner
Oct 05 13:27:24 devops-vm1 systemd[1]: java-app.service: Main process exited, code=killed, status=9/KILL
Oct 05 13:27:24 devops-vm1 systemd[1]: java-app.service: Failed with result 'signal'.
Oct 05 13:27:34 devops-vm1 systemd[1]: java-app.service: Scheduled restart job, restart counter is at 1.
Oct 05 13:27:34 devops-vm1 systemd[1]: Started java-app.service - Java Spring Boot Demo Application.
Oct 05 13:27:34 devops-vm1 sh[7312]: Application started under user java-runner
^C
```

**Phân tích nhật ký tự phục hồi:**
1. `13:27:24`: Tiến trình chính bị kill bất thường (`code=killed, status=9/KILL`), Systemd ghi nhận `Failed with result 'signal'`.
2. `13:27:24` -> `13:27:34`: Đúng 10 giây theo cấu hình `RestartSec=10`, Systemd lên lịch khởi động lại (`Scheduled restart job`).
3. `13:27:34`: Dịch vụ tự động phục hồi thành công với tiến trình mới (PID `7312`) tiếp tục chạy dưới user `java-runner`.
