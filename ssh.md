# Tài Liệu Hướng Dẫn SSH: Từ Căn Bản Đến Thực Hành Cấu Hình Server & Client

---

## 1. SSH Là Gì?

**SSH (Secure Shell)** là một giao thức mạng mật mã dùng để truy cập, quản trị và điều khiển máy chủ (VPS/Server) từ xa một cách an toàn thông qua các mạng không bảo mật (như Internet).

* **Port mặc định:** `22` (Sử dụng giao thức TCP).
* **Mô hình hoạt động:** Client - Server.
  * **SSH Client:** Ứng dụng trên máy cá nhân (`openssh-client`, Terminal, PowerShell...).
  * **SSH Server:** Dịch vụ chạy ngầm trên máy chủ (`sshd`).
* **Cơ chế bảo mật:** Toàn bộ dữ liệu truyền giữa Client và Server đều được mã hóa bằng thuật toán mã hóa đối xứng (sau khi hoàn tất bắt tay), giúp chống nghe lén (Eavesdropping) và can thiệp dữ liệu (Man-in-the-middle).

---

## 2. Các Phương Thức Xác Thực Trong SSH

### Cách 1: Xác Thực Bằng Mật Khẩu (Password Authentication)

Khi kết nối, SSH Server sẽ yêu cầu nhập mật khẩu của tài khoản người dùng trên Server.

* **Ưu điểm:** Dễ dùng, không cần thiết lập trước ở máy Client.
* **Nhược điểm:**
  * Dễ bị tấn công **Brute-force** hoặc **Dictionary Attack** (bot quét tự động thử hàng triệu mật khẩu).
  * Tốn thời gian nhập mật khẩu thủ công mỗi lần kết nối.

### Cách 2: Xác Thực Bằng SSH Key (Asymmetric Key Authentication)

Đây là phương thức chuẩn mực và an toàn nhất trên môi trường Production. Phương thức này dựa trên thuật toán **mã hóa bất đối xứng**.

Một cặp SSH Key gồm 2 phần:
* **Private Key (Khóa bí mật):** Lưu trữ an toàn tại máy cá nhân (Client), có phân quyền bảo mật chặt chẽ (`600`). **Tuyệt đối không chia sẻ file này.**
* **Public Key (Khóa công khai):** Được tải lên và lưu tại file `~/.ssh/authorized_keys` trên Server.

**Cơ chế hoạt động (Challenge-Response):**
1. **Client gửi Fingerprint:** Client lấy mã định danh (Key Fingerprint) của các Private Key đang có gửi sang Server.
2. **Server tìm Public Key:** Server đối chiếu Fingerprint với danh sách trong file `authorized_keys`. Nếu thấy khớp, Server dùng Public Key đó mã hóa một chuỗi dữ liệu ngẫu nhiên (Challenge/Nonce) rồi gửi lại cho Client.
3. **Client giải mã & Ký số:** Client dùng Private Key tương ứng giải mã lấy chuỗi ngẫu nhiên, sau đó kết hợp với Session ID và dùng Private Key **ký số (Sign)** lên dữ liệu này rồi gửi Chữ ký số (Response) về Server.
4. **Server xác thực:** Server dùng Public Key xác minh Chữ ký số. Nếu chữ ký hợp lệ, Server cấp quyền đăng nhập mà không cần mật khẩu.

---

## 3. Hướng Dẫn Thực Hành Cấu Hình Từ A - Z Bằng Lệnh Linux

1. **Cài đặt và bật dịch vụ SSH Server:**
   ```bash
   sudo apt update && sudo apt install openssh-server -y

   # Kích hoạt và kiểm tra trạng thái sshd
   sudo systemctl enable --now sshd
   sudo systemctl status sshd
   ```

2. **Các bug găp phải như là cài bridge:**
    - Nếu bridge (Automatic) thì trên thiết bị chúng ta có nhiều cổng kết nối thông tin nên nó sẽ chọn ngẫu nhiên một cái nếu nó k trúng cart mạng máy mình thì nó sẽ có tạo bridge được giữa card mạng với cái máy ảo. Nếu nối được máy ảo sẽ có ip trên router.
  
3. **Config ssh**
    - sudo nano /etc/ssh/sshd_config: 
      + PasswordAuthentication no
      + KbdInteractiveAuthentication no
      + PubkeyAuthentication yes
    - sudo sshd -t: kiểm tra cấu hỉnh có đúng hay k?.
    - sudo systemctl restart ssh: khởi động lại ssh.
    - nano ~/.ssh/authorized_keys: điền public key trong file .pub của client.
    - nếu cấu hình key xong lần sau ssh, thì k cần mật khẩu luôn.
    - Trên máy mình vào C:/Users/TeenUser/.ssh/ để xem private key, public key.