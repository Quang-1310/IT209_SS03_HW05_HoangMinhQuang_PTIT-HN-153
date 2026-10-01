## 1. Mục tiêu

- Lưu mã nguồn web tĩnh tại `/opt/my-app/html/`.
- Lưu access log và error log tại `/opt/my-app/logs/`.
- User `devops` quản lý mã nguồn mà không cần dùng `sudo` khi cập nhật file.
- Tiến trình Nginx (`www-data`) chỉ đọc nội dung web và được phép ghi log.

## 2. Cấu trúc thư mục

```text
/opt/my-app/
├── html/
│   └── index.html
└── logs/
    ├── access.log
    └── error.log
```

## 3. Tạo thư mục và phân quyền

Thực hiện trên Ubuntu/Debian bằng tài khoản có quyền `sudo`:

```bash
sudo install -d -o devops -g www-data -m 2750 /opt/my-app
sudo install -d -o devops -g www-data -m 2750 /opt/my-app/html
sudo install -d -o devops -g www-data -m 2770 /opt/my-app/logs

sudo install -o devops -g www-data -m 640 /dev/null /opt/my-app/html/index.html
sudo -u devops sh -c 'printf "%s\n" "<!doctype html><html><body><h1>My App</h1></body></html>" > /opt/my-app/html/index.html'

sudo touch /opt/my-app/logs/access.log /opt/my-app/logs/error.log
sudo chown devops:www-data /opt/my-app/logs/access.log /opt/my-app/logs/error.log
sudo chmod 660 /opt/my-app/logs/access.log /opt/my-app/logs/error.log
```

Nếu `/opt/my-app/` đã tồn tại, chuẩn hóa toàn bộ owner và quyền như sau:

```bash
sudo chown -R devops:www-data /opt/my-app
sudo find /opt/my-app/html -type d -exec chmod 2750 {} \;
sudo find /opt/my-app/html -type f -exec chmod 640 {} \;
sudo chmod 2750 /opt/my-app
sudo chmod 2770 /opt/my-app/logs
sudo find /opt/my-app/logs -type f -exec chmod 660 {} \;
```

Bit `setgid` trong quyền `2xxx` giúp file/thư mục mới kế thừa group `www-data`. Quyền thực tế:

| Đường dẫn | Owner | Quyền | Ý nghĩa |
|---|---|---:|---|
| `/opt/my-app` | `devops:www-data` | `2750` | `devops` toàn quyền; `www-data` đọc và đi qua thư mục |
| `/opt/my-app/html` | `devops:www-data` | `2750` | Nginx đọc nội dung web; user khác không truy cập |
| Tệp trong `html` | `devops:www-data` | `640` | `devops` đọc/ghi; Nginx chỉ đọc |
| `/opt/my-app/logs` | `devops:www-data` | `2770` | `devops` và `www-data` đều có thể ghi log |
| Tệp log | `devops:www-data` | `660` | `devops` và `www-data` đọc/ghi |

## 4. Cấu hình Nginx

Tệp nộp kèm: [`my-app.conf`](my-app.conf)

Cài đặt Server Block:

```bash
sudo cp my-app.conf /etc/nginx/sites-available/my-app
sudo ln -s /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/my-app
```

Nếu cấu hình mặc định cũng sử dụng `listen 80 default_server`, hãy tắt nó để tránh xung đột:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Kiểm tra và nạp lại cấu hình:

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl status nginx --no-pager
```

Kết quả mong đợi của `nginx -t`:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 5. Kiểm tra

### 5.1. Kiểm tra owner và permission

```bash
ls -la /opt/my-app/
ls -la /opt/my-app/html/ /opt/my-app/logs/
```

Kết quả mẫu (cần chụp kết quả thực tế trên máy chủ khi nộp):

```text
drwxr-s--- 4 devops www-data 4096 ... .
drwxr-s--- 2 devops www-data 4096 ... html
drwxrws--- 2 devops www-data 4096 ... logs

/opt/my-app/html/:
-rw-r----- 1 devops www-data ... index.html

/opt/my-app/logs/:
-rw-rw---- 1 devops www-data ... access.log
-rw-rw---- 1 devops www-data ... error.log
```

### 5.2. Kiểm tra ghi file bằng user `devops`

Đăng nhập bằng `devops`, không dùng `sudo`:

```bash
whoami
echo "Update Test" >> /opt/my-app/html/index.html
tail -n 2 /opt/my-app/html/index.html
```

Kết quả mong đợi:

```text
devops
</html>
Update Test
```

### 5.3. Kiểm tra Nginx và access log

Tạo request rồi đọc log:

```bash
curl -i http://127.0.0.1/
tail -n 5 /opt/my-app/logs/access.log
```

Để theo dõi log theo thời gian thực:

```bash
tail -f /opt/my-app/logs/access.log
```

Kết quả mong đợi: HTTP trả về `200 OK`, nội dung `index.html` được hiển thị và `access.log` xuất hiện một dòng request tương tự:

```text
127.0.0.1 - - [...] "GET / HTTP/1.1" 200 ... "-" "curl/..."
```

### 5.4. Kiểm tra trực tiếp quyền của Nginx

```bash
sudo -u www-data test -r /opt/my-app/html/index.html && echo "www-data: read HTML OK"
sudo -u www-data test -w /opt/my-app/html/index.html || echo "www-data: write HTML denied (expected)"
sudo -u www-data test -w /opt/my-app/logs && echo "www-data: write logs OK"
```

Kết quả mong đợi:

```text
www-data: read HTML OK
www-data: write HTML denied (expected)
www-data: write logs OK
```