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

## Hướng dẫn cài đặt

### 1. Tạo file docker-compose.yml

Tạo file `docker-compose.yml` với nội dung sau:

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

### 2. Khởi động services

```bash
# Khởi động các container
docker-compose up -d

# Kiểm tra trạng thái
docker-compose ps

# Xem logs
docker-compose logs -f
```

### 3. Truy cập ứng dụng

Mở trình duyệt và truy cập: `http://localhost:6003`

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

## Quản lý

### Dừng services

```bash
docker-compose down
```

### Dừng và xóa dữ liệu

```bash
docker-compose down -v
```

### Cập nhật lên phiên bản mới

```bash
docker-compose pull
docker-compose up -d
```

### Backup dữ liệu

```bash
# Backup database
docker exec pgvector pg_dump -U xiaozhi xiaozhi_mcphub > backup.sql

# Backup thư mục data
tar -czf appdata-backup.tar.gz appdata/
```

### Restore dữ liệu

```bash
# Restore database
cat backup.sql | docker exec -i pgvector psql -U xiaozhi -d xiaozhi_mcphub

# Restore thư mục data
tar -xzf appdata-backup.tar.gz
```

## Troubleshooting

### Lỗi port đã được sử dụng

Nếu port 5433 hoặc 6003 đã được sử dụng, thay đổi port mapping trong `docker-compose.yml`:

```yaml
ports:
  - "5434:5432"  # Đổi từ 5433 sang 5434
  # hoặc
  - "6004:3000"  # Đổi từ 6003 sang 6004
```

### Container không khởi động

```bash
# Xem logs chi tiết
docker-compose logs mcphub
docker-compose logs db

# Khởi động lại
docker-compose restart
```

### Database connection error

Đảm bảo service `db` đã sẵn sàng trước khi `mcphub` khởi động. Healthcheck đã được cấu hình để xử lý việc này.

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