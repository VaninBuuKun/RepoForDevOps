# Cài Đặt Docker Trên Ubuntu Server

## Mục tiêu

Cài đặt Docker Engine và Docker Compose trên Ubuntu Server để chuẩn bị deploy các service như Identity Service, Shipping Service, Redis, RabbitMQ, MySQL,...

---

## 1. Gỡ Docker cũ (nếu có)

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

---

## 2. Cập nhật hệ thống

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 3. Cài đặt các package cần thiết

```bash
sudo apt install -y ca-certificates curl gnupg
```

---

## 4. Thêm Docker GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

---

## 5. Thêm Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

---

## 6. Cài Docker Engine

```bash
sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io \
docker-buildx-plugin docker-compose-plugin
```

---

## 7. Kiểm tra Docker

Kiểm tra phiên bản:

```bash
docker --version
```

Ví dụ:

```text
Docker version 28.x.x
```

Kiểm tra trạng thái service:

```bash
sudo systemctl status docker
```

Kết quả mong đợi:

```text
active (running)
```

---

## 8. Chạy thử container đầu tiên

```bash
sudo docker run hello-world
```

Nếu xuất hiện:

```text
Hello from Docker!
```

Docker đã được cài đặt thành công.

---

## 9. Cho phép chạy Docker không cần sudo

Thêm user hiện tại vào group docker:

```bash
sudo usermod -aG docker $USER
```

Đăng xuất và SSH lại:

```bash
exit
```

Kiểm tra:

```bash
docker ps
```

Nếu chạy được mà không cần `sudo` là thành công.

---

## 10. Kiểm tra Docker Compose

```bash
docker compose version
```

Ví dụ:

```text
Docker Compose version v2.x.x
```

Lưu ý:

```bash
docker compose
```

✅ Lệnh mới

```bash
docker-compose
```

⚠️ Lệnh cũ, có thể không còn được cài mặc định.

---

## 11. Test Docker Compose

Tạo file `docker-compose.yml`:

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "7000:80"
```

Khởi động:

```bash
docker compose up -d
```

Kiểm tra container:

```bash
docker ps
```

Truy cập:

```text
http://192.168.1.12:7000
```

Nếu xuất hiện trang **Welcome to nginx!** thì Docker và Docker Compose đã hoạt động bình thường.

---

## Các lệnh Docker thường dùng

### Danh sách container đang chạy

```bash
docker ps
```

### Danh sách toàn bộ container

```bash
docker ps -a
```

### Xem logs

```bash
docker logs <container_name>
```

### Theo dõi logs realtime

```bash
docker logs -f <container_name>
```

### Dừng container

```bash
docker stop <container_name>
```

### Khởi động lại container

```bash
docker restart <container_name>
```

### Xóa container

```bash
docker rm <container_name>
```

### Danh sách image

```bash
docker images
```

### Xóa image

```bash
docker rmi <image_name>
```

---

## Bước tiếp theo

Sau khi hoàn tất:

* Deploy Identity Service bằng Docker
* Deploy Shipping Service bằng Docker
* Tạo Docker Compose cho nhiều service
* Thêm YARP API Gateway
* Cấu hình Nginx Reverse Proxy
* Thiết lập HTTPS với Let's Encrypt
* Xây dựng CI/CD bằng GitHub Actions
