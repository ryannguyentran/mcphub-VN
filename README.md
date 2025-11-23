# XiaoZhi MCPHub Docker Deployment

Hướng dẫn triển khai XiaoZhi MCPHub sử dụng Docker và Docker Compose.

## Giới thiệu

XiaoZhi MCPHub là một nền tảng quản lý MCP (Model Context Protocol) servers, cho phép bạn dễ dàng triển khai và quản lý các MCP servers thông qua giao diện web.

## Yêu cầu hệ thống

- Docker Engine 20.10+
- Docker Compose v2.0+
- Port 5433 và 6003 chưa được sử dụng

## Cấu trúc thư mục

```
.
├── docker-compose.yml
├── pgdata/              # Dữ liệu PostgreSQL (tự động tạo)
├── appdata/             # Dữ liệu ứng dụng (tự động tạo)
├── scripts/             # Scripts tùy chỉnh
└── mcp-servers/         # Cấu hình MCP servers
```

## Hướng dẫn cài đặt chi tiết trên Ubuntu

### Bước 1: Cập nhật hệ thống

```bash
# Cập nhật danh sách package
sudo apt update

# Nâng cấp các package đã cài
sudo apt upgrade -y
```

### Bước 2: Cài đặt Docker

```bash
# Cài đặt các package cần thiết
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Thêm GPG key của Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Thêm Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Cập nhật lại package list
sudo apt update

# Cài đặt Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Kiểm tra Docker đã cài thành công
sudo docker --version
```

### Bước 3: Cài đặt Docker Compose

```bash
# Tải Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Cấp quyền thực thi
sudo chmod +x /usr/local/bin/docker-compose

# Kiểm tra Docker Compose đã cài thành công
docker-compose --version
```

### Bước 4: Cấu hình Docker (không cần sudo)

```bash
# Thêm user hiện tại vào group docker
sudo usermod -aG docker $USER

# Áp dụng thay đổi group (hoặc logout/login lại)
newgrp docker

# Kiểm tra có chạy docker không cần sudo không
docker ps
```

### Bước 5: Tạo thư mục project

```bash
# Tạo thư mục cho project
mkdir -p ~/xiaozhi-mcphub

# Di chuyển vào thư mục
cd ~/xiaozhi-mcphub

# Tạo các thư mục cần thiết
mkdir -p scripts mcp-servers
```

### Bước 6: Tạo file docker-compose.yml

```bash
# Tạo file docker-compose.yml bằng nano
nano docker-compose.yml
```

Sao chép nội dung sau vào file (Ctrl+Shift+V để paste):

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
    ports:
      - "5433:5432"
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U xiaozhi -d xiaozhi_mcphub -h localhost"]
      interval: 5s
      timeout: 3s
      retries: 20
      
  mcphub:
    image: huangjunsen/xiaozhi-mcphub:latest
    container_name: xiaozhi-mcphub
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    ports:
      - "6003:3000"
    environment:
      DATABASE_URL: "postgres://xiaozhi:xiaozhi123456@db:5432/xiaozhi_mcphub"
      NODE_ENV: production
      SMART_ROUTING_ENABLED: "false"
      # JWT_SECRET: "nên đặt random cho production"
      # BASE_PATH: "/mcphub"
      # OPENAI_API_KEY: "sk-........"
    volumes:
      - ./appdata:/app/data
      - ./scripts:/scripts
      - ./mcp-servers:/mcp-servers
      - /var/run/docker.sock:/var/run/docker.sock
```

**Lưu file:**
- Nhấn `Ctrl + O` để lưu
- Nhấn `Enter` để xác nhận
- Nhấn `Ctrl + X` để thoát

### Bước 7: Kiểm tra file đã tạo

```bash
# Xem nội dung file vừa tạo
cat docker-compose.yml

# Kiểm tra cấu trúc thư mục
ls -la
```

### Bước 8: Tải Docker images

```bash
# Tải các image cần thiết (optional, docker-compose sẽ tự tải)
docker pull pgvector/pgvector:pg16
docker pull huangjunsen/xiaozhi-mcphub:latest
```

### Bước 9: Khởi động services

```bash
# Khởi động các container ở chế độ background
docker-compose up -d

# Chờ khoảng 30 giây để database khởi động hoàn tất
```

### Bước 10: Kiểm tra trạng thái

```bash
# Kiểm tra các container đang chạy
docker-compose ps

# Hoặc xem chi tiết hơn
docker ps

# Xem logs của tất cả services
docker-compose logs

# Xem logs realtime (Ctrl+C để thoát)
docker-compose logs -f

# Xem logs của một service cụ thể
docker-compose logs mcphub
docker-compose logs db
```

### Bước 11: Kiểm tra kết nối

```bash
# Kiểm tra port đang mở
sudo netstat -tlnp | grep -E '(5433|6003)'

# Hoặc dùng ss
ss -tlnp | grep -E '(5433|6003)'

# Test kết nối database
docker exec pgvector psql -U xiaozhi -d xiaozhi_mcphub -c "SELECT version();"
```

### Bước 12: Truy cập ứng dụng

Mở trình duyệt và truy cập:
- **Local**: `http://localhost:6003`
- **Từ máy khác**: `http://IP_CUA_SERVER:6003`

**Kiểm tra IP của server:**
```bash
# Xem IP của server
hostname -I
# hoặc
ip addr show
```

### Bước 13: Cấu hình Firewall (nếu cần)

```bash
# Nếu dùng UFW
sudo ufw allow 6003/tcp
sudo ufw allow 5433/tcp

# Reload firewall
sudo ufw reload

# Kiểm tra status
sudo ufw status
```

## Cấu hình nâng cao

### Biến môi trường

Bạn có thể tùy chỉnh các biến môi trường sau trong service `mcphub`:

| Biến | Mô tả | Mặc định |
|------|-------|----------|
| `DATABASE_URL` | URL kết nối PostgreSQL | `postgres://xiaozhi:xiaozhi123456@db:5432/xiaozhi_mcphub` |
| `NODE_ENV` | Môi trường chạy | `production` |
| `SMART_ROUTING_ENABLED` | Bật/tắt smart routing | `false` |
| `JWT_SECRET` | Secret key cho JWT (khuyến nghị đổi) | - |
| `BASE_PATH` | Base path cho ứng dụng | - |
| `OPENAI_API_KEY` | API key OpenAI (nếu cần) | - |

### Thay đổi mật khẩu database

**Quan trọng**: Trong môi trường production, bạn nên thay đổi mật khẩu mặc định:

1. Đổi `POSTGRES_PASSWORD` trong service `db`
2. Cập nhật `DATABASE_URL` trong service `mcphub` với mật khẩu mới

### Sử dụng JWT Secret

Để tăng cường bảo mật, uncomment và đặt giá trị ngẫu nhiên cho `JWT_SECRET`:

```bash
# Tạo JWT secret ngẫu nhiên
openssl rand -base64 32
```

## Các lệnh quản lý thường dùng

### Xem logs

```bash
# Xem logs realtime của tất cả services
docker-compose logs -f

# Xem logs của MCPHub
docker-compose logs -f mcphub

# Xem logs của Database
docker-compose logs -f db

# Xem 100 dòng logs cuối
docker-compose logs --tail=100

# Xem logs từ 10 phút trước
docker-compose logs --since 10m
```

### Khởi động lại services

```bash
# Khởi động lại tất cả
docker-compose restart

# Khởi động lại service cụ thể
docker-compose restart mcphub
docker-compose restart db
```

### Dừng services

```bash
# Dừng tất cả services (giữ nguyên data)
docker-compose stop

# Dừng và xóa containers (giữ nguyên volumes/data)
docker-compose down

# Dừng và xóa tất cả (bao gồm volumes/data)
docker-compose down -v
```

### Khởi động lại sau khi dừng

```bash
# Khởi động services đã dừng
docker-compose start

# Hoặc khởi động từ đầu
docker-compose up -d
```

### Cập nhật lên phiên bản mới

```bash
# Di chuyển vào thư mục project
cd ~/xiaozhi-mcphub

# Dừng services hiện tại
docker-compose down

# Tải phiên bản mới nhất
docker-compose pull

# Khởi động lại với phiên bản mới
docker-compose up -d

# Xem logs để kiểm tra
docker-compose logs -f
```

### Backup dữ liệu

```bash
# Tạo thư mục backup
mkdir -p ~/backups

# Backup database
docker exec pgvector pg_dump -U xiaozhi xiaozhi_mcphub > ~/backups/mcphub_$(date +%Y%m%d_%H%M%S).sql

# Backup thư mục appdata
cd ~/xiaozhi-mcphub
tar -czf ~/backups/appdata_$(date +%Y%m%d_%H%M%S).tar.gz appdata/

# Backup thư mục pgdata (database files)
tar -czf ~/backups/pgdata_$(date +%Y%m%d_%H%M%S).tar.gz pgdata/

# Xem các file backup
ls -lh ~/backups/
```

### Restore dữ liệu

```bash
# Restore database từ file SQL
cat ~/backups/mcphub_YYYYMMDD_HHMMSS.sql | docker exec -i pgvector psql -U xiaozhi -d xiaozhi_mcphub

# Restore appdata
cd ~/xiaozhi-mcphub
tar -xzf ~/backups/appdata_YYYYMMDD_HHMMSS.tar.gz

# Khởi động lại services sau khi restore
docker-compose restart
```

### Xóa hoàn toàn và cài lại

```bash
# Di chuyển vào thư mục project
cd ~/xiaozhi-mcphub

# Dừng và xóa containers, networks, volumes
docker-compose down -v

# Xóa thư mục data local
rm -rf pgdata/ appdata/

# Khởi động lại từ đầu
docker-compose up -d
```

### Kiểm tra tài nguyên sử dụng

```bash
# Xem CPU, RAM của containers
docker stats

# Xem dung lượng disk của containers
docker system df

# Xem chi tiết dung lượng của từng container
docker ps -s
```

### Truy cập vào container

```bash
# Truy cập vào container mcphub
docker exec -it xiaozhi-mcphub sh

# Truy cập vào container database
docker exec -it pgvector bash

# Truy cập PostgreSQL CLI
docker exec -it pgvector psql -U xiaozhi -d xiaozhi_mcphub

# Thoát khỏi container
exit
```

### Dọn dẹp Docker

```bash
# Xóa các container đã dừng
docker container prune -f

# Xóa các image không dùng
docker image prune -a -f

# Xóa các volume không dùng
docker volume prune -f

# Xóa tất cả (containers, images, volumes, networks không dùng)
docker system prune -a --volumes -f
```

## Troubleshooting - Xử lý lỗi thường gặp

### Lỗi 1: Port đã được sử dụng

**Triệu chứng:**
```
Error: bind: address already in use
```

**Cách khắc phục:**

```bash
# Kiểm tra port nào đang chiếm dụng
sudo lsof -i :6003
sudo lsof -i :5433

# Hoặc dùng netstat
sudo netstat -tlnp | grep 6003
sudo netstat -tlnp | grep 5433

# Kill process đang dùng port (thay PID bằng số hiện ra)
sudo kill -9 PID

# Hoặc đổi port trong docker-compose.yml
nano docker-compose.yml
# Đổi "6003:3000" thành "6004:3000"
# Đổi "5433:5432" thành "5434:5432"

# Khởi động lại
docker-compose up -d
```

### Lỗi 2: Container không khởi động

**Cách kiểm tra:**

```bash
# Xem logs để tìm lỗi
docker-compose logs mcphub
docker-compose logs db

# Xem trạng thái chi tiết
docker ps -a

# Xem logs 200 dòng cuối
docker-compose logs --tail=200
```

**Cách khắc phục:**

```bash
# Thử khởi động lại
docker-compose restart

# Nếu không được, dừng và xóa rồi tạo lại
docker-compose down
docker-compose up -d

# Xem logs realtime
docker-compose logs -f
```

### Lỗi 3: Database connection error

**Triệu chứng:**
```
Error: connect ECONNREFUSED
FATAL: password authentication failed
```

**Cách khắc phục:**

```bash
# Kiểm tra database đã sẵn sàng chưa
docker exec pgvector pg_isready -U xiaozhi

# Kiểm tra có kết nối được không
docker exec pgvector psql -U xiaozhi -d xiaozhi_mcphub -c "SELECT 1;"

# Xem logs database
docker-compose logs db

# Reset database
docker-compose down
docker volume rm xiaozhi-mcphub_pgdata  # Hoặc tên volume của bạn
docker-compose up -d
```

### Lỗi 4: Permission denied

**Triệu chứng:**
```
permission denied while trying to connect to the Docker daemon socket
```

**Cách khắc phục:**

```bash
# Thêm user vào group docker
sudo usermod -aG docker $USER

# Logout và login lại, hoặc chạy
newgrp docker

# Hoặc tạm thời dùng sudo
sudo docker-compose up -d
```

### Lỗi 5: Không tải được Docker image

**Triệu chứng:**
```
Error response from daemon: pull access denied
```

**Cách khắc phục:**

```bash
# Kiểm tra kết nối internet
ping -c 3 google.com

# Thử tải image thủ công
docker pull pgvector/pgvector:pg16
docker pull huangjunsen/xiaozhi-mcphub:latest

# Xem logs chi tiết
docker-compose up

# Khởi động lại Docker daemon
sudo systemctl restart docker
```

### Lỗi 6: Out of disk space

**Cách kiểm tra:**

```bash
# Kiểm tra dung lượng disk
df -h

# Xem dung lượng Docker sử dụng
docker system df
```

**Cách khắc phục:**

```bash
# Dọn dẹp Docker
docker system prune -a --volumes -f

# Xóa logs cũ
sudo find /var/lib/docker/containers/ -name "*-json.log" -exec truncate -s 0 {} \;

# Xóa các image không dùng
docker image prune -a -f
```

### Lỗi 7: Cannot connect to localhost:6003

**Cách kiểm tra:**

```bash
# Kiểm tra container có chạy không
docker ps | grep mcphub

# Kiểm tra port có mở không
sudo netstat -tlnp | grep 6003

# Test từ server
curl http://localhost:6003
```

**Cách khắc phục:**

```bash
# Kiểm tra firewall
sudo ufw status
sudo ufw allow 6003/tcp

# Nếu truy cập từ máy khác, kiểm tra IP
hostname -I

# Thử restart container
docker-compose restart mcphub
```

### Lỗi 8: Docker Compose not found

**Cách khắc phục:**

```bash
# Cài lại Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

# Kiểm tra
docker-compose --version

# Hoặc dùng lệnh mới
docker compose version
```

### Lỗi 9: Healthcheck failed

**Cách kiểm tra:**

```bash
# Xem health status
docker ps

# Xem logs healthcheck
docker inspect pgvector | grep -A 10 Health
```

**Cách khắc phục:**

```bash
# Chờ thêm thời gian (database cần 30-60s để khởi động)
sleep 60

# Kiểm tra lại
docker ps

# Xem logs database
docker-compose logs db

# Restart nếu cần
docker-compose restart db
```

### Lỗi 10: Volume mount error

**Triệu chứng:**
```
Error response from daemon: invalid mount config
```

**Cách khắc phục:**

```bash
# Kiểm tra quyền thư mục
ls -la ~/xiaozhi-mcphub/

# Tạo lại thư mục nếu cần
mkdir -p ~/xiaozhi-mcphub/{pgdata,appdata,scripts,mcp-servers}

# Cấp quyền
sudo chmod -R 755 ~/xiaozhi-mcphub/

# Khởi động lại
docker-compose down
docker-compose up -d
```

## Tài nguyên

- Repository gốc: [https://github.com/huangjunsen0406/xiaozhi-mcphub](https://github.com/huangjunsen0406/xiaozhi-mcphub)
- Docker Hub: [huangjunsen/xiaozhi-mcphub](https://hub.docker.com/r/huangjunsen/xiaozhi-mcphub)

## Đóng góp

Mọi đóng góp đều được hoan nghênh! Vui lòng tạo issue hoặc pull request trên repository gốc.

## License

Vui lòng tham khảo license từ repository gốc.

## Cảm ơn

Cảm ơn tác giả [@huangjunsen0406](https://github.com/huangjunsen0406) đã phát triển XiaoZhi MCPHub - một công cụ tuyệt vời cho việc quản lý MCP servers.

Dự án gốc: [https://github.com/huangjunsen0406/xiaozhi-mcphub](https://github.com/huangjunsen0406/xiaozhi-mcphub)