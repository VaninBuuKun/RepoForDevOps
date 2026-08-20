# Nginx – Vai Trò & Công Dụng Trong Deploy Web

> Tài liệu tổng hợp dành cho người đang học deploy Backend (BE) và Frontend (FE).

---

## 1. Nginx là gì?

**Nginx** (đọc là "engine-x") là một phần mềm mã nguồn mở, ban đầu được viết ra để làm **web server** (giống Apache), nhưng qua thời gian nó đã trở thành công cụ đa năng, đảm nhận rất nhiều vai trò khác trong hệ thống deploy hiện đại.

Điểm mạnh cốt lõi: Nginx dùng kiến trúc **event-driven, non-blocking** (bất đồng bộ), nên xử lý được **hàng chục nghìn kết nối đồng thời** mà tốn rất ít RAM/CPU — đây là lý do nó gần như là mặc định trong mọi hệ thống production ngày nay.

```
Apache: mỗi request -> tạo 1 thread/process riêng  => tốn tài nguyên khi traffic lớn
Nginx : 1 worker process xử lý hàng ngàn request cùng lúc bằng event loop
```

---

## 2. Kiến trúc hoạt động (tổng quan)

```
                    ┌─────────────────────┐
                    │   Nginx Master      │  <- đọc config, quản lý worker
                    │      Process        │
                    └─────────┬───────────┘
                              │ spawn
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐    ┌──────────┐
        │ Worker 1 │   │ Worker 2 │    │ Worker N │
        └──────────┘   └──────────┘    └──────────┘
        mỗi worker xử lý hàng ngàn connection bằng event loop (epoll/kqueue)
```

- **Master process**: đọc file cấu hình, quản lý worker, reload config không downtime.
- **Worker process**: xử lý request thực tế (I/O, proxy, static file...).

---

## 3. Các vai trò chính của Nginx

### 3.1. Web Server — Phục vụ file tĩnh (Static File Server)

Nginx cực nhanh trong việc trả về HTML, CSS, JS, ảnh... Đây là lý do khi build FE (React/Vue/Angular) xong, ta thường dùng Nginx để "serve" thư mục `build/` hoặc `dist/`.

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;  # hỗ trợ SPA routing
    }
}
```

### 3.2. Reverse Proxy — Cổng trung gian tới Backend

Đây là vai trò **quan trọng nhất** khi deploy full-stack. Thay vì client gọi trực tiếp vào server Node.js/Django/Spring Boot (thường chạy ở port nội bộ như 3000, 8000...), client sẽ gọi vào Nginx, rồi Nginx **chuyển tiếp (proxy)** request đó tới backend.

```
Client (browser)
      │  https://example.com/api/users
      ▼
┌───────────┐        proxy_pass         ┌──────────────────┐
│   Nginx   │ ────────────────────────► │  Backend (Node)   │
│ (port 80) │ ◄──────────────────────── │  (port 3000)       │
└───────────┘                           └──────────────────┘
```

```nginx
server {
    listen 80;
    server_name api.example.com;

    location /api/ {
        proxy_pass http://localhost:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Lợi ích của reverse proxy:**
- Ẩn cấu trúc hạ tầng backend thật (bảo mật).
- Cho phép chạy nhiều app/backend trên cùng 1 server (80/443) bằng cách phân theo domain hoặc path.
- Tập trung xử lý SSL, nén, cache tại 1 điểm.

### 3.3. Load Balancer — Cân bằng tải

Khi backend có nhiều instance (scale ngang), Nginx phân phối request đều ra các server để tránh 1 server quá tải.

```nginx
upstream backend_servers {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend_servers;
    }
}
```

Các thuật toán phổ biến: `round-robin` (mặc định), `least_conn` (ưu tiên server ít kết nối nhất), `ip_hash` (giữ session theo IP client).

### 3.4. SSL/TLS Termination — Xử lý HTTPS

Nginx thường là nơi "giải mã" HTTPS, sau đó giao tiếp nội bộ với backend qua HTTP thường (nhanh hơn, đơn giản hơn).

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
    }
}

# Redirect HTTP -> HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### 3.5. Caching — Tăng tốc độ phản hồi

Nginx có thể cache response từ backend để giảm tải, tăng tốc cho các request lặp lại (đặc biệt hữu ích với API ít thay đổi hoặc static asset).

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g;

server {
    location /api/ {
        proxy_cache my_cache;
        proxy_cache_valid 200 10m;
        proxy_pass http://localhost:3000;
    }
}
```

### 3.6. Nén dữ liệu (Gzip/Brotli) — Giảm băng thông

```nginx
gzip on;
gzip_types text/css application/javascript application/json;
gzip_min_length 1024;
```

### 3.7. Rate Limiting — Chống spam/DDoS cơ bản

```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=mylimit burst=20 nodelay;
        proxy_pass http://localhost:3000;
    }
}
```

---

## 4. Mô hình deploy FE + BE điển hình với Nginx

```
                        Internet
                            │
                            ▼
                     ┌─────────────┐
                     │    Nginx    │  <- port 80/443 (duy nhất mở ra ngoài)
                     └──────┬──────┘
             ┌───────────────┴───────────────┐
             ▼                                ▼
   location / (FE static)             location /api/ (reverse proxy)
   serve React/Vue build               proxy_pass -> Backend (Node/Django...)
             │                                │
   /var/www/frontend/dist          Backend chạy trên port nội bộ (vd: 3000)
                                    (không expose ra internet trực tiếp)
```

Đây chính là mô hình phổ biến nhất khi bạn deploy 1 VPS (DigitalOcean, AWS EC2, Vultr...): Nginx đứng trước, FE build ra static file được Nginx serve trực tiếp, còn mọi request `/api/*` được Nginx forward vào backend chạy ngầm (thường quản lý bởi PM2, systemd, hoặc Docker container).

---

## 5. Nginx vs Apache — So sánh nhanh

| Tiêu chí | Nginx | Apache |
|---|---|---|
| Kiến trúc | Event-driven, async | Process/thread-based |
| Hiệu năng static file | Rất nhanh | Trung bình |
| Xử lý nhiều connection | Rất tốt (hàng chục nghìn) | Kém hơn khi traffic cao |
| Cấu hình | Đơn giản, tập trung | Linh hoạt qua `.htaccess` |
| Vai trò phổ biến | Reverse proxy, load balancer, static server | Web server truyền thống |

Thực tế nhiều hệ thống dùng **cả hai**: Nginx đứng trước làm reverse proxy/load balancer, Apache xử lý logic phía sau (hoặc thay Apache bằng backend app server như Node/Gunicorn).

---

## 6. Các lệnh quản lý cơ bản

```bash
sudo nginx -t                     # test cú pháp config trước khi apply
sudo systemctl restart nginx      # khởi động lại
sudo systemctl reload nginx       # reload config, KHÔNG downtime
sudo systemctl status nginx       # xem trạng thái
tail -f /var/log/nginx/error.log  # xem log lỗi realtime
tail -f /var/log/nginx/access.log # xem log truy cập
```

Cấu trúc thư mục config phổ biến (Ubuntu/Debian):
```
/etc/nginx/nginx.conf              # config chính
/etc/nginx/sites-available/        # nơi chứa các file config site
/etc/nginx/sites-enabled/          # symlink tới site đang active
```

---

## 7. Lỗi thường gặp khi mới học

- **502 Bad Gateway**: Nginx không kết nối được tới backend (backend chưa chạy, sai port, sai `proxy_pass`).
- **413 Request Entity Too Large**: cần tăng `client_max_body_size` (vd khi upload file).
- **CORS lỗi dù đã proxy**: nhớ kiểm tra header `Access-Control-Allow-Origin` — nếu FE và BE cùng domain qua Nginx thì thường không cần CORS nữa.
- **Config sai khiến Nginx không khởi động được**: luôn chạy `nginx -t` trước khi reload.

---

## 8. Lộ trình học tiếp theo (Recommend)

Vì bạn đang học deploy FE + BE, đây là các mảnh ghép nên học tiếp theo, xếp theo thứ tự hợp lý:

1. **Domain & DNS cơ bản** — hiểu A record, CNAME, cách trỏ domain về IP VPS.
2. **SSL miễn phí với Let's Encrypt (Certbot)** — tự động cấp và gia hạn HTTPS, tích hợp thẳng với Nginx.
3. **Process Manager cho Backend**: PM2 (Node.js) hoặc systemd service — giữ backend luôn chạy, tự restart khi crash, chạy khi reboot server.
4. **Docker & Docker Compose** — đóng gói FE, BE, database thành container, deploy đồng bộ, dễ tái tạo môi trường. Rất nên học sau khi nắm Nginx cơ bản, vì thường Nginx cũng được chạy trong 1 container làm "gateway".
5. **CI/CD cơ bản** (GitHub Actions/GitLab CI) — tự động build & deploy mỗi khi push code, thay vì deploy tay.
6. **Environment variables & secrets management** — không hardcode API key, DB password.
7. **Database deploy & backup** — PostgreSQL/MySQL/MongoDB chạy production, cấu hình backup định kỳ.
8. **Monitoring & Logging** — công cụ như UptimeRobot (miễn phí, đơn giản), hoặc nâng cao hơn: Prometheus + Grafana.
9. **Security headers cơ bản** trong Nginx: `X-Frame-Options`, `X-Content-Type-Options`, `Content-Security-Policy`.
10. **Container orchestration nâng cao** (khi hệ thống lớn hơn): Docker Swarm hoặc Kubernetes — không cần thiết ngay lúc này, chỉ nên biết tới khi hệ thống có nhiều service.

### Gợi ý thứ tự thực hành cụ thể
```
Bước 1: Deploy 1 VPS (Ubuntu) -> cài Nginx -> serve 1 trang HTML tĩnh
Bước 2: Deploy backend Node.js chạy port 3000 bằng PM2
Bước 3: Cấu hình Nginx reverse proxy /api/ -> backend
Bước 4: Build FE (React/Vue) -> copy vào /var/www -> Nginx serve
Bước 5: Trỏ domain thật + cài SSL bằng Certbot
Bước 6: Đóng gói toàn bộ bằng Docker Compose
Bước 7: Thiết lập CI/CD tự động deploy khi push lên GitHub
```

---

## Tổng kết

Nginx không chỉ là "web server" — trong hệ thống deploy thực tế, nó đóng vai trò như một **cánh cổng trung tâm (gateway)**: nhận toàn bộ traffic từ internet, quyết định request nào đi tới FE (file tĩnh), request nào đi tới BE (API), xử lý HTTPS, cân bằng tải, cache, và bảo vệ hệ thống phía sau. Nắm vững Nginx là một trong những kỹ năng nền tảng quan trọng nhất khi làm DevOps/deploy.