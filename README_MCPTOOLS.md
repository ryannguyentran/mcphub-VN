# HƯỚNG DẪN TẢI VÀ CÀI ĐẶT SCRIPTS CHO XIAOZHI MCP HUB

[![Release](https://img.shields.io/badge/Release-mcphub-blue?logo=github)](https://github.com/ryannguyentran/mcphub-VN/releases/tag/mcphub)
[![Scripts](https://img.shields.io/badge/Scripts-Available-green)](https://github.com/ryannguyentran/mcphub-VN)

> **Hướng dẫn tải và cài đặt các script tiếng Việt cho XiaoZhi MCP Hub**

---

## 📦 BƯỚC 1: TẢI FILE SCRIPTS

### Cách 1: Tải trực tiếp từ GitHub Releases

1. Truy cập link sau:
   ```
   https://github.com/ryannguyentran/mcphub-VN/releases/tag/mcphub
   ```

2. Tại trang Releases, tìm phần **Assets** và tải file nén (thường có tên dạng `scripts.zip` hoặc `mcphub-scripts.zip`)

3. Hoặc click trực tiếp vào các file bạn cần tải

### Cách 2: Dùng lệnh wget/curl (Trên Linux/Ubuntu)

```bash
# Di chuyển vào thư mục dự án
cd xiaozhi-mcphub

# Tải file scripts (thay URL bằng link file thực tế từ releases)
wget https://github.com/ryannguyentran/mcphub-VN/releases/download/mcphub/5tools.zip
# Hoặc dùng curl
curl -L -O https://github.com/ryannguyentran/mcphub-VN/releases/download/mcphub/5tools.zip
```

---

## 📂 BƯỚC 2: TẠO THƯ MỤC SCRIPTS

Trước khi giải nén, đảm bảo thư mục `scripts` tồn tại:

```bash
# Di chuyển vào thư mục dự án
cd xiaozhi-mcphub

# Tạo thư mục scripts nếu chưa có
mkdir -p scripts
```

---

## 📥 BƯỚC 3: GIẢI NÉN FILE VÀO THƯ MỤC SCRIPTS

### Trên Linux/Ubuntu:

```bash
# Giải nén file zip vào thư mục scripts
sudo unzip 5tools.zip -d scripts/

# Hoặc nếu file là tar.gz
tar -xzf 5tools.tar.gz -C scripts/
```

### Trên Windows:

1. Click phải vào file `scripts.zip`
2. Chọn **"Extract All..."** hoặc **"Giải nén tất cả..."**
3. Chọn đường dẫn đích: `xiaozhi-mcphub\scripts\`
4. Click **"Extract"** hoặc **"Giải nén"**

### Trên macOS:

```bash
# Giải nén file zip
unzip 5tools.zip -d scripts/

# Hoặc double-click file zip trong Finder và di chuyển vào thư mục scripts
```

---

## ✅ BƯỚC 4: KIỂM TRA CẤU TRÚC THƯ MỤC

Sau khi giải nén, cấu trúc thư mục sẽ như sau:

```
xiaozhi-mcphub/
├── docker-compose.yml
├── scripts/
│   ├── script1.js
│   ├── script2.py
│   ├── config.json
│   └── ...
└── data/
```

Kiểm tra bằng lệnh:

```bash
# Xem danh sách file trong thư mục scripts
ls -la scripts/

# Hoặc trên Windows (PowerShell)
dir scripts\
```

---

## 🔄 BƯỚC 5: CẬP NHẬT VÀ KHỞI ĐỘNG LẠI

Sau khi thêm scripts, khởi động lại container để áp dụng thay đổi:

```bash
# Khởi động lại container
sudo docker compose restart

# Hoặc dừng và khởi động lại hoàn toàn
sudo docker compose down
sudo docker compose up -d
# Kiểm tra scripts đã tồn tại trong container hay chưa
sudo docker exec <idscontainer> ls -l /app/scripts
```

---

## 🔍 KIỂM TRA SCRIPTS ĐÃ HOẠT ĐỘNG

Xem logs để đảm bảo scripts được load thành công:

```bash
# Xem logs của container mcphub
sudo docker compose logs -f mcphub
```

Tìm các dòng log liên quan đến việc load scripts, ví dụ:
```
✓ Loaded script: script1.js
✓ Loaded script: script2.py
✓ All scripts initialized successfully
```

---

## 🔧 XỬ LÝ LỖI THƯỜNG GẶP

### Lỗi: "Permission denied" khi giải nén

**Giải pháp:**
```bash
# Thêm sudo trước lệnh
sudo unzip scripts.zip -d scripts/

# Cấp quyền cho thư mục scripts
sudo chmod -R 755 scripts/
```

### Lỗi: Scripts không được load

**Giải pháp:**
1. Kiểm tra đường dẫn trong `docker-compose.yml` có mapping volume đúng không:
   ```yaml
   volumes:
     - ./scripts:/app/scripts
   ```

2. Khởi động lại container:
   ```bash
   sudo docker compose restart
   ```

### Lỗi: File bị giải nén sai cấu trúc

**Giải pháp:**
```bash
# Xóa thư mục scripts và tạo lại
rm -rf scripts/
mkdir scripts/

# Giải nén lại đúng cách
unzip scripts.zip -d scripts/
```

---

## 📚 THÔNG TIN THÊM

### Cập nhật Scripts mới

Khi có phiên bản scripts mới:

```bash
# Tải phiên bản mới từ releases
wget https://github.com/ryannguyentran/mcphub-VN/releases/download/mcphub/scripts-v2.zip

# Backup scripts cũ
mv scripts/ scripts-backup/

# Giải nén scripts mới
mkdir scripts/
unzip scripts-v2.zip -d scripts/

# Khởi động lại
sudo docker compose restart
```

### Xóa cache nếu cần

```bash
# Xóa container và tạo lại (giữ nguyên data)
sudo docker compose down
sudo docker compose up -d --force-recreate
```

---

## 🔗 LIÊN KẾT HỮU ÍCH

- **Repository Scripts:** [https://github.com/ryannguyentran/mcphub-VN](https://github.com/ryannguyentran/mcphub-VN)
- **Releases Page:** [https://github.com/ryannguyentran/mcphub-VN/releases/tag/mcphub](https://github.com/ryannguyentran/mcphub-VN/releases/tag/mcphub)
- **Repository Chính:** [https://github.com/huangjunsen0406/xiaozhi-mcphub](https://github.com/huangjunsen0406/xiaozhi-mcphub)

---

## 🙏 LỜI CẢM ƠN

Cảm ơn bạn đã sử dụng các scripts tiếng Việt cho XiaoZhi MCP Hub!

Nếu gặp vấn đề, hãy tạo Issue tại: [GitHub Issues](https://github.com/ryannguyentran/mcphub-VN/issues)

---

**Chúc bạn sử dụng thành công! 🚀**

*Cập nhật: 2025*
