md_content = """# HƯỚNG DẪN CHI TIẾT VỀ NGINX & MÔ HÌNH DEPLOY THỰC TẾ TRÊN UBUNTU SERVER

Tài liệu này cung cấp kiến thức toàn diện, chi tiết và thực chiến về **Nginx**, cách hoạt động của Nginx khi kết hợp với **Frontend (FE)** và **Backend (BE)**, cùng các bước cấu hình chuẩn Production trên hệ điều hành **Ubuntu Server**.

---

## MỤC LỤC
1. [Nginx Là Gì & Tại Sao Lại Dùng Nginx?](#1-nginx-là-gì--tại-sao-lại-dùng-nginx)
2. [Sự Khác Biệt Giữa Frontend Và Backend Khi Deploy](#2-sự-khác-biệt-giữa-frontend-và-backend-khi-deploy)
3. [Mô Hình Kiến Trúc & Luồng Đi Của Request](#3-mô-hình-kiến-trúc--luồng-đi-của-request)
4. [Cấu Trúc Thư Mục Nginx Trên Ubuntu Server](#4-cấu-trúc-thư-mục-nginx-trên-ubuntu-server)
5. [Hướng Dẫn Cấu Hình Thực Tế (Từng Bước)](#5-hướng-dẫn-cấu-hình-thực-tế-từng-bước)
   - [5.1. Chuẩn bị thư mục & Build Frontend](#51-chuẩn-bị-thư-mục--build-frontend)
   - [5.2. Chạy ứng dụng Backend dưới dạng Systemd Service](#52-chạy-ứng-dụng-backend-dưới-dạng-systemd-service)
   - [5.3. Tạo File Cấu Hình Nginx Hoàn Chỉnh](#53-tạo-file-cấu-hình-nginx-hoàn-chỉnh)
   - [5.4. Kích hoạt và Reload Nginx](#54-kích-hoạt-và-reload-nginx)
6. [Cấu Hình HTTPS Miễn Phí Với Let's Encrypt (Certbot)](#6-cấu-hình-https-miễn-phí-với-lets-encrypt-certbot)
7. [Giải Thích Từng Directive Trong File Cấu Hình](#7-giải-thích-từng-directive-trong-file-cấu-hình)
8. [Các Lỗi Thường Gặp & Cách Khắc Phục (Troubleshooting)](#8-các-lỗi-thường-gặp--cách-khắc-phục-troubleshooting)
9. [Bảng Lệnh Cheat Sheet Quản Lý Nginx](#9-bảng-lệnh-cheat-sheet-quản-lý-nginx)

---

## 1. Nginx Là Gì & Tại Sao Lại Dùng Nginx?

### 1.1. Định nghĩa
**Nginx** (phát âm: *Engine-X*) là một phần mềm mã nguồn mở chạy trên Linux/Windows đóng vai trò là:
- **Web Server:** Phục vụ các file tĩnh (HTML, CSS, JS, hình ảnh, video).
- **Reverse Proxy:** Nhận request từ Client và chuyển tiếp (forward) đến các ứng dụng Backend nội bộ.
- **Load Balancer:** Phân phối tải giữa nhiều server Backend.
- **HTTP Cache:** Lưu bản sao phản hồi để giảm tải cho Backend.

### 1.2. Tại sao Nginx lại cực kỳ phổ biến?
Khác với Apache truyền thống sử dụng kiến trúc *Thread-based* (mỗi request tạo một thread riêng, tốn nhiều RAM/CPU), Nginx sử dụng kiến trúc **Event-driven (Bất đồng bộ - Asynchronous)**. 
- **Tối ưu tài nguyên:** Một process của Nginx có thể xử lý hàng chục nghìn kết nối đồng thời (C10K problem) mà chỉ tiêu tốn rất ít RAM và CPU.
- **Bảo mật:** Giúp ẩn thông tin và IP nội bộ của Backend khỏi người dùng ngoài Internet.

---

## 2. Sự Khác Biệt Giữa Frontend Và Backend Khi Deploy

Khi deploy ứng dụng Web hiện đại (như React, Vue, Angular kết hợp với .NET, Node.js, Spring Boot, Python):

| Tiêu chí | Frontend (SPA: React / Vue / Vite) | Backend (.NET / Node.js / Python / Java) |
| :--- | :--- | :--- |
| **Bản chất sau build** | Là tập hợp các file tĩnh (`.html`, `.js`, `.css`, `.png`). | Là chương trình/dịch vụ cần **chạy liên tục** (Runtime Process) để nhận và xử lý logic. |
| **Cách Nginx xử lý** | Nginx **đọc trực tiếp** từ đĩa cứng Ubuntu (`/var/www/...`) và gửi về Trình duyệt. | Nginx đóng vai trò **Reverse Proxy**, chuyển request HTTP tới Port nội bộ mà Backend đang chạy. |
| **Port hoạt động** | Không dùng Port riêng, nằm trực tiếp trên Port 80/443 của Nginx. | Chạy ở một Port nội bộ ẩn (ví dụ: `127.0.0.1:5000` hoặc `127.0.0.1:8080`). |

---

## 3. Mô Hình Kiến Trúc & Luồng Đi Của Request

### 3.1. Sơ đồ tổng thể trên Ubuntu Server

```text
                                Internet / Client (Browser)
                                             │
                                             │ (HTTP: Port 80 / HTTPS: Port 443)
                                             ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ UBUNTU SERVER                                                                          │
│                                                                                        │
│   ┌────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                  NGINX                                         │   │
│   │                    (Cổng giao tiếp duy nhất với bên ngoài)                     │   │
│   └───────────────┬────────────────────────────────────────────────┬───────────────┘   │
│                   │                                                │                   │
│                   │ (1) Request lấy Giao diện / Trang web          │ (2) Request API   │
│                   │     Path: / hoặc /about, /dashboard            │     Path: /api/*  │
│                   ▼                                                ▼                   │
│   ┌───────────────────────────────────────────────┐  ┌─────────────────────────────┐   │
│   │ Static Storage (Ổ cứng Ubuntu)                │  │ Reverse Proxy Forward       │   │
│   │ Path: /var/www/myapp/frontend/dist            │  └──────────────┬──────────────┘   │
│   │ Files: index.html, main.js, style.css         │                 │                  │
│   └───────────────────────────────────────────────┘                 │                  │
│                                                                     ▼                  │
│                                                      ┌─────────────────────────────┐   │
│                                                      │ Backend Process (Systemd)   │   │
│                                                      │ .NET / Node.js / Python     │   │
│                                                      │ Address: 127.0.0.1:5000     │   │
│                                                      └─────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────────────────┘