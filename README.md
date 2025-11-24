# HƯỚNG DẪN TRIỂN KHAI XIAOZHI MCP HUB TRÊN UBUNTU

[![Docker](https://img.shields.io/badge/Docker-Required-blue?logo=docker)](https://www.docker.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-Compatible-orange?logo=ubuntu)](https://ubuntu.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

> **Dành cho người mới bắt đầu - Hướng dẫn từ A đến Z**
> 
***Account mặc định MCP HUB: admin / admin123
> 
Tài liệu này hướng dẫn chi tiết cách cài đặt môi trường và triển khai ứng dụng **XiaoZhi MCP Hub** theo 2 phương pháp:

1. **Network Host** - Chạy trực tiếp trên mạng máy chủ (Dễ kết nối, hiệu năng cao) ⚡
2. **Network Bridge** - Chạy qua mạng ảo Docker (Cách chuẩn, dễ quản lý port) 🔒

---

## 📋 MỤC LỤC

- [Phần 1: Cài đặt Docker](#phần-1-cài-đặt-docker-bắt-buộc)
- [Phần 2: Deploy dùng Network Host](#phần-2-deploy-dùng-network-host-khuyên-dùng)
- [Phần 3: Chuyển về Network Bridge](#phần-3-cách-chuyển-về-network-bridge)
- [Phần 4: Các lệnh quản trị](#phần-4-các-lệnh-quản-trị-cần-biết)
- [Xử lý lỗi thường gặp](#-xử-lý-lỗi-thường-gặp)
- [Lời cảm ơn](#-lời-cảm-ơn)

---

## PHẦN 1: CÀI ĐẶT DOCKER (BẮT BUỘC)

Trước khi bắt đầu, bạn cần cài Docker lên VPS/Máy chủ Ubuntu của mình.

### Bước 1: Cập nhật hệ thống

Mở Terminal và chạy lệnh:

```bash
sudo apt update && sudo apt upgrade -y
```

### Bước 2: Cài đặt Docker và Docker Compose

```bash
sudo apt install docker.io docker-compose-plugin -y
```

### Bước 3: Kiểm tra cài đặt

Chạy lệnh sau, nếu hiện ra phiên bản (ví dụ: `Docker Compose version v2...`) là thành công:

```bash
docker compose version
```

### Bước 4: Cấp quyền cho user hiện tại (Tùy chọn)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## PHẦN 2: DEPLOY DÙNG "NETWORK HOST" (KHUYÊN DÙNG)

Cách này giúp ứng dụng sử dụng trực tiếp IP của máy chủ, không bị chặn bởi lớp mạng ảo của Docker.

### 1. Tạo thư mục dự án

```bash
mkdir xiaozhi-mcphub
cd xiaozhi-mcphub
```

### 2. Tạo file cấu hình

Sử dụng trình soạn thảo nano:

```bash
nano docker-compose.yml
```

Copy và Paste nội dung dưới đây vào file:

```yaml
version: "3.8"

volumes:
  pgdata:
  appdata:

services:
  db:
    image: pgvector/pgvector:pg16
    container_name: pgvector
    restart: unless-stopped
    environment:
      POSTGRES_USER: xiaozhi
      POSTGRES_PASSWORD: xiaozhi123456
      POSTGRES_DB: xiaozhi_mcphub
    # Mở port database ra localhost để App (đang chạy host mode) kết nối được
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U xiaozhi -d xiaozhi_mcphub -h 127.0.0.1"]
      interval: 5s
      timeout: 3s
      retries: 20

  mcphub:
    image: huangjunsen/xiaozhi-mcphub:latest
    container_name: xiaozhi-mcphub
    # QUAN TRỌNG: Chế độ mạng Host
    network_mode: "host"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    environment:
      # Vì chạy Host mode, app phải gọi DB qua IP localhost của máy chủ
      DATABASE_URL: "postgres://xiaozhi:xiaozhi123456@127.0.0.1:5432/xiaozhi_mcphub"
      NODE_ENV: production
      SMART_ROUTING_ENABLED: "false"
      # Ép ứng dụng lắng nghe mọi IP để truy cập được từ bên ngoài
      HOST: "0.0.0.0"
      PORT: "3000"
      # JWT_SECRET: "hay-thay-doi-chuoi-nay-khi-chay-that"
    volumes:
      - appdata:/app/data
      - ./scripts:/app/scripts
```

**Lưu file:** Nhấn `Ctrl+O` → `Enter` để ghi, và `Ctrl+X` để thoát.

### 3. Khởi chạy

```bash
sudo docker compose up -d
```

### 4. Mở Tường lửa (Firewall)

Nếu bạn không truy cập được web, hãy mở port 3000:

```bash
sudo ufw allow 3000/tcp
```

> **Lưu ý:** Nếu dùng VPS Google Cloud/AWS/Azure, nhớ mở thêm **Security Group** trên web console.

### 5. Truy cập ứng dụng

Mở trình duyệt và truy cập:

```
http://IP_CỦA_BẠN:3000
```

---

## PHẦN 3: CÁCH CHUYỂN VỀ "NETWORK BRIDGE"

Nếu bạn gặp lỗi xung đột port hoặc muốn chạy theo chuẩn Docker (cô lập mạng), hãy sửa lại file như sau.

### 1. Sửa file cấu hình

```bash
nano docker-compose.yml
```

Xóa nội dung cũ và thay bằng nội dung chuẩn Bridge dưới đây:

```yaml
version: "3.8"

volumes:
  pgdata:
  appdata:

services:
  db:
    image: pgvector/pgvector:pg16
    container_name: pgvector
    restart: unless-stopped
    environment:
      POSTGRES_USER: xiaozhi
      POSTGRES_PASSWORD: xiaozhi123456
      POSTGRES_DB: xiaozhi_mcphub
    # Bridge mode: Database không cần expose port ra ngoài cũng được
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U xiaozhi -d xiaozhi_mcphub -h 127.0.0.1"]
      interval: 5s
      timeout: 3s
      retries: 20

  mcphub:
    image: huangjunsen/xiaozhi-mcphub:latest
    container_name: xiaozhi-mcphub
    # XÓA dòng network_mode: "host"
    # THÊM dòng ports để ánh xạ port
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    environment:
      # Bridge mode: App gọi DB qua tên service "db"
      DATABASE_URL: "postgres://xiaozhi:xiaozhi123456@db:5432/xiaozhi_mcphub"
      NODE_ENV: production
      SMART_ROUTING_ENABLED: "false"
    volumes:
      - appdata:/app/data
      - ./scripts:/app/scripts
```

### 2. Cập nhật lại container

Lệnh này sẽ buộc Docker xóa container cũ và tạo container mới theo cấu hình vừa sửa:

```bash
sudo docker compose up -d --force-recreate
```

---

## PHẦN 4: CÁC LỆNH QUẢN TRỊ CẦN BIẾT

Dưới đây là các lệnh bạn sẽ dùng hàng ngày. Hãy chạy chúng trong thư mục `xiaozhi-mcphub`.

### 1. Xem nhật ký (Logs) để sửa lỗi

```bash
sudo docker compose logs -f
```

*(Nhấn `Ctrl+C` để thoát)*

### 2. Khởi động lại toàn bộ

```bash
sudo docker compose restart
```

### 3. Tắt và Xóa Container (Giữ lại dữ liệu)

```bash
sudo docker compose down
```

### 4. Xóa SẠCH SÀNH SANH (Xóa cả dữ liệu Database)

⚠️ **Cảnh báo:** Dữ liệu sẽ mất hết.

```bash
sudo docker compose down -v
```

### 5. Xem trạng thái container

```bash
sudo docker compose ps
```

### 6. Cập nhật image mới nhất

```bash
sudo docker compose pull
sudo docker compose up -d
```

---

## 🔧 XỬ LÝ LỖI THƯỜNG GẶP

### Lỗi: "Cannot connect to database"

**Giải pháp:**
- Kiểm tra container database đã chạy chưa: `sudo docker compose ps`
- Xem logs database: `sudo docker compose logs db`
- Đợi thêm 10-20 giây để database khởi động hoàn toàn

### Lỗi: "Port 3000 already in use"

**Giải pháp:**
- Tìm process đang dùng port: `sudo lsof -i :3000`
- Dừng process đó hoặc đổi port trong file `docker-compose.yml`

### Lỗi: "Permission denied"

**Giải pháp:**
- Thêm `sudo` trước các lệnh docker
- Hoặc thêm user vào group docker (xem Phần 1, Bước 4)

---

## 📚 TÀI LIỆU THAM KHẢO

- [Docker Documentation](https://docs.docker.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [PGVector Extension](https://github.com/pgvector/pgvector)

---

## 🙏 LỜI CẢM ƠN

Xin chân thành cảm ơn tác giả **Huang Junsen** đã phát triển và chia sẻ **XiaoZhi MCP Hub** - một công cụ tuyệt vời cho cộng đồng.

🔗 **Repository gốc:** [https://github.com/huangjunsen0406/xiaozhi-mcphub](https://github.com/huangjunsen0406/xiaozhi-mcphub)

Nếu bạn thấy dự án này hữu ích, hãy cho tác giả một ⭐ trên GitHub!

---

## 📝 GIẤY PHÉP

Tài liệu này được chia sẻ miễn phí cho mục đích học tập và phi lợi nhuận.

---

## 🤝 ĐÓNG GÓP

Nếu bạn phát hiện lỗi hoặc muốn cải thiện tài liệu này, hãy tạo Pull Request hoặc Issue trên GitHub.

---

**Chúc bạn triển khai thành công! 🚀**

*Cập nhật lần cuối: 2025*
