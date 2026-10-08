# Bài 4: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## Thông tin học viên

| Trường | Nội dung |
|--------|----------|
| **Họ và tên** | [Họ Tên Học Viên] |
| **Mã lớp** | [Mã Lớp] |
| **Môn học** | IT209 – Triển khai ứng dụng |
| **Bài tập** | Session 07 – Exercise 4 |

---

## Mục tiêu

Cấu hình Nginx làm **Reverse Proxy** sử dụng _Path Matching_:

- `GET /` → phục vụ trang tĩnh từ `/var/www/html/`
- `GET /api/*` → chuyển tiếp đến Spring Boot backend chạy tại `http://127.0.0.1:8082/`

---

## Cấu trúc tệp nộp bài

```
homework/session_07/ex4/
├── spring-proxy.conf   # Cấu hình Nginx Virtual Server Block
├── index.html          # Trang tĩnh đặt tại /var/www/html/index.html
└── README.md           # Báo cáo này
```

---

## Hướng dẫn triển khai từng bước

### Bước 1 – Tạo thư mục & đặt trang tĩnh

```bash
sudo mkdir -p /var/www/html
sudo cp index.html /var/www/html/index.html
sudo chown -R www-data:www-data /var/www/html
```

### Bước 2 – Sao chép tệp cấu hình vào sites-available

```bash
sudo cp spring-proxy.conf /etc/nginx/sites-available/spring-proxy.conf
```

### Bước 3 – Kích hoạt cấu hình bằng symlink

```bash
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf \
           /etc/nginx/sites-enabled/spring-proxy.conf
```

### Bước 4 – Kiểm tra cú pháp Nginx

```bash
sudo nginx -t
```

**Kết quả mong đợi:**

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### Bước 5 – Reload Nginx để áp dụng cấu hình

```bash
sudo systemctl reload nginx
```

---

## Kết quả kiểm tra

### 1. Kiểm tra cú pháp cấu hình

```bash
sudo nginx -t
```

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

### 2. Kiểm tra trang tĩnh (HTTP 200)

```bash
curl -I http://localhost/
```

```
HTTP/1.1 200 OK
Server: nginx/1.x.x
Date: ...
Content-Type: text/html
Content-Length: ...
Connection: keep-alive
```

### 3. Kiểm tra Reverse Proxy đến Spring Boot

```bash
curl -I http://localhost/api/health
```

```
HTTP/1.1 200 OK
Server: nginx/1.x.x
Date: ...
Content-Type: application/json
...
```

> Nếu Spring Boot chưa chạy, `/api/health` trả về **502 Bad Gateway** –  
> hãy khởi động ứng dụng trước: `java -jar app.jar --server.port=8082`

---

## Giải thích cấu hình chính

```nginx
# Static: phục vụ file từ /var/www/html/
location / {
    root  /var/www/html;
    index index.html index.htm;
    try_files $uri $uri/ =404;
}

# Dynamic: chuyển tiếp sang Spring Boot :8082
location /api/ {
    proxy_pass http://127.0.0.1:8082/;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

| Tiêu đề | Ý nghĩa |
|---------|---------|
| `Host` | Giữ nguyên host gốc từ client |
| `X-Real-IP` | IP thực của client gửi đến backend |
| `X-Forwarded-For` | Chuỗi IP proxy chain |
| `X-Forwarded-Proto` | Giao thức gốc (http/https) |

---

## Lưu ý quan trọng

- `proxy_pass http://127.0.0.1:8082/;` — dấu `/` cuối URL **rất quan trọng**: Nginx sẽ cắt bỏ phần `/api/` khỏi URI trước khi chuyển tiếp, tránh đường dẫn bị nhân đôi.
- Nếu đã có cấu hình `default` trong `sites-enabled`, hãy xoá symlink của nó để tránh xung đột cổng 80: `sudo rm /etc/nginx/sites-enabled/default`
