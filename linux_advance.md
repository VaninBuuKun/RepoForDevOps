# Linux Commands Cho Dân Deploy — Cấp Độ Trung Bình

> Không nói lại mấy lệnh file cơ bản (`ls`, `cd`, `cp`, `mv`...). Đây là tập lệnh bạn **thực sự dùng hàng ngày** khi quản lý server, deploy app, debug production.

---

## 1. `systemctl` — Quản lý service (quan trọng nhất)

Mọi backend chạy production (Node, .NET, Django, Nginx, Docker...) đều nên được quản lý như 1 **service** của systemd — để nó tự chạy lại khi crash, tự start khi reboot server.

```bash
sudo systemctl start myapp        # khởi động service
sudo systemctl stop myapp         # dừng service
sudo systemctl restart myapp      # restart (dừng hẳn rồi chạy lại)
sudo systemctl reload myapp       # reload config, không downtime (nếu service hỗ trợ)
sudo systemctl status myapp       # xem trạng thái, log gần nhất
sudo systemctl enable myapp       # tự chạy khi server reboot
sudo systemctl disable myapp      # tắt tự chạy khi reboot
sudo systemctl daemon-reload      # load lại sau khi sửa file .service
```

Tự viết 1 service cho app của bạn tại `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My Node Backend
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/node server.js
Restart=on-failure
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp   # enable + start cùng lúc
```

> **Đây chính là thay thế "chuyên nghiệp" cho PM2** khi bạn muốn mọi thứ quản lý đồng bộ bằng systemd thay vì thêm 1 tool ngoài.

---

## 2. Quản lý tiến trình (Process)

```bash
ps aux                     # liệt kê toàn bộ tiến trình đang chạy
ps aux | grep node         # tìm tiến trình Node đang chạy
top                        # theo dõi CPU/RAM realtime (tool có sẵn)
htop                       # bản đẹp hơn của top (cần cài: apt install htop)

kill <PID>                 # gửi tín hiệu terminate (SIGTERM) - graceful
kill -9 <PID>               # force kill (SIGKILL) - dùng khi process treo cứng
pkill -f "node server.js"  # kill theo tên process (không cần PID)

nohup node server.js &     # chạy app, không bị kill khi đóng terminal/SSH
disown                     # gỡ process con khỏi shell hiện tại, chạy độc lập
jobs                       # xem các job đang chạy nền trong session
fg %1                      # đưa job về foreground
bg %1                      # đẩy job chạy nền

sudo lsof -i :5027         # Tìm các tiến trình chạy trên port 5027
sudo kill -9 <PID>         # Xóa cưỡng chế process port 5027.

```

**Khi nào dùng gì:**
- `kill` trước, `kill -9` chỉ khi process không phản hồi.
- `nohup ... &` là cách "chữa cháy" nhanh, còn production thật thì nên dùng `systemctl` hoặc PM2/Docker.

---

## 3. Theo dõi tài nguyên hệ thống

```bash
free -h                    # xem RAM đang dùng/còn trống (-h = human readable)
df -h                      # xem dung lượng ổ đĩa còn lại theo từng phân vùng
du -sh /var/www/*          # xem thư mục nào đang chiếm nhiều dung lượng nhất
uptime                     # server đã chạy bao lâu + load average
vmstat 1                   # thống kê CPU/memory/IO mỗi giây (debug hiệu năng)
lsblk                      # liệt kê các ổ đĩa/phân vùng
```

> Khi server bị full disk (lỗi rất hay gặp khi deploy — do log, do Docker image tồn đọng): `df -h` để biết ổ nào đầy, `du -sh` để tìm thủ phạm.

---

## 4. Network & Debug kết nối

```bash
ss -tulnp                  # xem các port đang lắng nghe (thay thế netstat hiện đại)
netstat -tulnp              # tương tự (cần cài net-tools trên 1 số distro mới)
lsof -i :3000               # xem process nào đang chiếm port 3000
curl -I https://example.com # kiểm tra header response, mã status
curl -v http://localhost:3000/api/health   # debug API chi tiết (verbose)
wget <url>                  # tải file từ internet về server
ping google.com             # kiểm tra kết nối mạng cơ bản
traceroute example.com      # xem gói tin đi qua các chặng nào (debug mạng chậm)
dig example.com             # tra cứu DNS record (A, CNAME, MX...)
nslookup example.com        # tương tự dig, đơn giản hơn
```

**Case thực tế hay gặp:** app không start được vì "port đã bị chiếm"
```bash
lsof -i :3000        # tìm PID đang giữ port 3000
kill -9 <PID>         # giải phóng port
```

---

## 5. Firewall

```bash
sudo ufw status                  # xem trạng thái firewall (Ubuntu)
sudo ufw allow 80/tcp             # mở port 80 (HTTP)
sudo ufw allow 443/tcp            # mở port 443 (HTTPS)
sudo ufw allow OpenSSH            # luôn nhớ mở SSH trước khi enable ufw!
sudo ufw enable                   # bật firewall
sudo ufw delete allow 8080        # đóng port đã mở
```

> **Lỗi kinh điển của người mới**: bật `ufw enable` mà quên `allow OpenSSH` trước → tự khóa mình khỏi server, phải vào bằng console của nhà cung cấp VPS để sửa.

---

## 6. Logs — `journalctl` & log file

```bash
journalctl -u myapp              # xem log của 1 service cụ thể (do systemd quản lý)
journalctl -u myapp -f           # theo dõi log realtime (giống tail -f)
journalctl -u myapp --since "1 hour ago"
journalctl -xe                   # xem log lỗi gần nhất toàn hệ thống

tail -f /var/log/nginx/error.log     # theo dõi log Nginx realtime
tail -n 200 /var/log/nginx/access.log | less
```

---

## 7. Quản lý package (apt / dnf)

```bash
sudo apt update                  # cập nhật danh sách package (Ubuntu/Debian)
sudo apt upgrade                 # nâng cấp package đã cài
sudo apt install nginx           # cài package
sudo apt remove nginx            # gỡ package
sudo apt autoremove              # dọn package rác không còn dùng

sudo dnf install nginx           # tương đương trên CentOS/RHEL/Fedora
```

---

## 8. Biến môi trường (Environment Variables)

```bash
export NODE_ENV=production        # set biến môi trường cho session hiện tại
echo $NODE_ENV                    # xem giá trị biến
env                                # liệt kê toàn bộ biến môi trường hiện có
unset NODE_ENV                     # xóa biến

# set vĩnh viễn cho user hiện tại:
echo 'export NODE_ENV=production' >> ~/.bashrc
source ~/.bashrc                   # load lại config ngay không cần mở terminal mới
```

> Trong production thật, thường dùng file `.env` + thư viện đọc env (dotenv) hoặc set trực tiếp trong file `.service` của systemd — **tránh hardcode secret vào code**.

---

## 9. Cron — Lên lịch chạy tự động

```bash
crontab -e                 # mở file cron của user hiện tại để chỉnh sửa
crontab -l                 # xem các cron job hiện có
```

Cú pháp: `phút giờ ngày tháng thứ command`

```bash
0 2 * * * /usr/bin/certbot renew --quiet     # 2h sáng mỗi ngày, gia hạn SSL
0 0 * * 0 rm -rf /var/log/myapp/*.log        # mỗi Chủ Nhật, dọn log cũ
*/5 * * * * curl -s http://localhost:3000/health > /dev/null   # health check mỗi 5 phút
```

---

## 10. SSH & Kết nối remote

```bash
ssh user@ip_server                    # kết nối SSH vào server
ssh -i mykey.pem user@ip_server       # kết nối bằng SSH key riêng
scp file.txt user@ip:/path/           # copy file từ máy local lên server
scp -r ./dist user@ip:/var/www/       # copy cả thư mục
ssh-keygen -t ed25519                  # tạo cặp key SSH (khuyên dùng ed25519 thay RSA)
ssh-copy-id user@ip_server             # đẩy public key lên server để login không cần password
```

---

## 11. `tmux` / `screen` — Giữ session chạy khi mất kết nối SSH

Rất quan trọng khi bạn chạy 1 lệnh lâu (build, migrate DB, deploy script) mà không muốn nó bị ngắt khi SSH bị disconnect.

```bash
tmux new -s deploy          # tạo session tên "deploy"
# ... chạy lệnh bất kỳ, kể cả tốn thời gian ...
# Ctrl+B rồi nhấn D          -> detach, thoát ra mà session vẫn chạy nền
tmux attach -t deploy       # quay lại session sau
tmux ls                     # xem các session đang có
tmux kill-session -t deploy # xóa session
```

---

## 12. Đóng gói/giải nén cho deploy artifact

```bash
tar -czvf app.tar.gz ./dist          # nén thư mục thành .tar.gz
tar -xzvf app.tar.gz -C /var/www/    # giải nén ra thư mục đích
zip -r app.zip ./dist                 # nén dạng zip
unzip app.zip -d /var/www/            # giải nén zip
```

---

## 13. User & quyền cho service (không phải lệnh file cơ bản)

```bash
sudo adduser deploy                   # tạo user riêng để chạy app (không nên chạy app bằng root!)
sudo usermod -aG docker deploy        # thêm user vào group docker (để chạy docker không cần sudo)
sudo -u deploy node server.js         # chạy lệnh dưới quyền user khác
groups deploy                         # xem user thuộc group nào
```

> **Nguyên tắc bảo mật cơ bản**: không bao giờ chạy app production bằng user `root`. Luôn tạo user riêng (`deploy`, `www-data`...) chỉ có quyền vừa đủ.

---

## 14. Bonus — Tiện ích tăng tốc thao tác

```bash
history | grep docker        # tìm lại lệnh docker đã gõ trước đó trong lịch sử
watch -n 2 "df -h"           # tự động chạy lại lệnh mỗi 2 giây (theo dõi realtime)
alias ll='ls -alF'           # tạo alias cho lệnh hay dùng (thêm vào ~/.bashrc)
!!                            # chạy lại lệnh vừa gõ trước đó
sudo !!                       # chạy lại lệnh trước với quyền sudo (khi quên sudo)
```

---

## Bảng tóm tắt nhanh — "Khi gặp vấn đề X, dùng lệnh Y"

| Tình huống | Lệnh |
|---|---|
| App không chạy, muốn xem lỗi | `journalctl -u myapp -f` |
| Port bị chiếm | `lsof -i :3000` |
| Server đầy ổ đĩa | `df -h` → `du -sh *` |
| RAM/CPU cao bất thường | `htop` |
| Kiểm tra app có phản hồi không | `curl -I http://localhost:PORT` |
| Chạy build lâu, sợ mất SSH | `tmux new -s build` |
| Firewall chặn port | `ufw allow <port>/tcp` |
| Muốn app tự restart khi crash | viết `.service` + `systemctl enable --now` |
| SSL sắp hết hạn | `crontab -e` để tự renew |

---

## Recommend học tiếp theo

Bạn đã có nền Linux command + Nginx (từ tài liệu trước), lộ trình hợp lý tiếp theo:

1. **Docker** — gần như bắt buộc. Học `docker build`, `docker run`, `docker-compose up -d`, `docker logs -f`, `docker exec -it`. Khi biết Docker, nhiều lệnh systemctl/process ở trên vẫn cần (để quản lý chính Docker daemon và container), nhưng việc deploy app sẽ đơn giản và nhất quán hơn nhiều.
2. **Docker Compose** — để chạy đồng thời FE + BE + Database + Nginx chỉ với 1 file `docker-compose.yml`.
3. **Reverse proxy Nginx trong Docker** — kết hợp 2 kiến thức bạn vừa học, rất phổ biến trong thực tế (Nginx làm gateway đứng ngoài, các service khác chạy trong container riêng).
4. **CI/CD (GitHub Actions)** — tự động hoá: mỗi lần push code, tự SSH vào server, pull code mới, build lại Docker image, restart service — thay vì làm tay từng bước ở trên.
5. Nếu bạn dùng **.NET**: học cách publish app dạng self-contained hoặc framework-dependent (`dotnet publish -c Release`), sau đó chạy bằng `systemctl` service như ví dụ ở mục 1 (Kestrel server đứng sau Nginx reverse proxy).
6. Nếu bạn dùng **React**: nhớ rằng React chỉ cần `npm run build` ra thư mục static (`build/` hoặc `dist/`) rồi để Nginx serve — **không cần Node chạy 24/7 cho FE**, khác hoàn toàn với BE.
7. **Monitoring nâng cao** khi hệ thống lớn dần: `htop`/`df` tay sẽ không đủ, lúc đó học thêm Prometheus + Grafana, hoặc đơn giản hơn là dịch vụ như UptimeRobot.

---

### Ghi nhớ cốt lõi

> Deploy = **chạy app bền vững (systemctl)** + **quản lý được nó khi có sự cố (process, log, network)** + **bảo vệ nó (firewall, user riêng, SSL)** + **tự động hoá (cron, CI/CD)**. Toàn bộ tài liệu này chính là công cụ cho 4 việc đó.