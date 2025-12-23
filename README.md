## Kiến trúc
- **Database Server**: MySQL server riêng biệt
- **Web Server**: Flask app + Nginx + Gunicorn

## Các bước deploy

### 1. Deploy Database Server

```bash
# Tạo instance từ OpenStack dashboard với:
# - Image: Ubuntu 22.04
# - User data: database_cloudinint.yaml
# - Security group: Mở port 22 (SSH) và 3306 (MySQL)
```

**Lưu ý IP của database server** (ví dụ: `192.168.1.100`)

### 2. Cập nhật Web Server Cloud-Init

Trước khi deploy web server, **BẮT BUỘC** sửa file `webserver_cloudinit.yaml`:

```yaml
# Tìm và thay thế TẤT CẢ các dòng có CHANGE_TO_DB_IP
# Thành IP thực của database server

# Dòng 1: Trong supervisor config (line ~38)
environment=DB_HOST="192.168.1.100",DB_USER="flaskuser",...

# Dòng 2: Trong init_db.sh (line ~55)
DB_HOST="${DB_HOST:-192.168.1.100}"

# Dòng 3: Trong runcmd (line ~71)
- export DB_HOST=192.168.1.100
```

### 3. Deploy Web Server

```bash
# Tạo instance từ OpenStack dashboard với:
# - Image: Ubuntu 22.04
# - User data: webserver_cloudinit.yaml (ĐÃ SỬA IP)
# - Security group: Mở port 22 (SSH) và 80 (HTTP)
# - Đảm bảo web server có thể kết nối tới database server
```
