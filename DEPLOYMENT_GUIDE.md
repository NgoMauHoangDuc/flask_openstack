# Flask App Deployment Guide - 2-Tier Architecture

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

## Kiểm tra deployment

### Database Server

SSH vào database server:
```bash
ssh dbadmin@<DB_SERVER_IP>

# Kiểm tra MySQL đang chạy
sudo systemctl status mysql

# Kiểm tra database đã tạo
mysql -u flaskuser -pFlaskPass123 -e "SHOW DATABASES;"

# Kiểm tra bind-address
sudo grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf
# Kết quả phải là: bind-address = 0.0.0.0
```

### Web Server

SSH vào web server:
```bash
ssh flaskuser@<WEB_SERVER_IP>

# Kiểm tra app đã clone
ls -la /opt/flask_app

# Kiểm tra kết nối database
mysql -h <DB_SERVER_IP> -u flaskuser -pFlaskPass123 -e "SELECT 1;"

# Kiểm tra supervisor
sudo supervisorctl status
# Kết quả: flask_app RUNNING

# Xem log nếu có lỗi
sudo tail -f /var/log/flask_app.err.log
sudo tail -f /var/log/flask_app.out.log

# Kiểm tra nginx
sudo systemctl status nginx
```

### Truy cập ứng dụng

Mở browser: `http://<WEB_SERVER_IP>`

## Troubleshooting

### Lỗi "Can't connect to MySQL server"

```bash
# Từ web server, kiểm tra kết nối
telnet <DB_SERVER_IP> 3306

# Nếu không kết nối được, kiểm tra:
# 1. Security group của DB server có mở port 3306 không
# 2. bind-address trong MySQL config
sudo grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf
```

### Lỗi supervisor không start được

```bash
# Xem log chi tiết
sudo supervisorctl tail flask_app stderr

# Restart manual
sudo supervisorctl restart flask_app
```

### Database tables chưa được tạo

```bash
cd /opt/flask_app
export DB_HOST=<DB_SERVER_IP>
export DB_USER=flaskuser
export DB_PASS=FlaskPass123
export DB_NAME=flaskdb

sudo -u flaskuser -E venv/bin/python -c "from app import app, db; app.app_context().push(); db.create_all()"
```

## Cấu hình Security Groups

### Database Server Security Group
- **Inbound Rules**:
  - SSH (22): From your IP
  - MySQL (3306): From Web Server security group hoặc subnet

### Web Server Security Group
- **Inbound Rules**:
  - SSH (22): From your IP  
  - HTTP (80): From 0.0.0.0/0 (public)
- **Outbound Rules**:
  - All traffic (để kết nối tới DB server và internet)

## Environment Variables

App sử dụng các biến môi trường sau:
- `DB_HOST`: IP/hostname của database server (default: localhost)
- `DB_USER`: MySQL username (default: flaskuser)
- `DB_PASS`: MySQL password (default: FlaskPass123)
- `DB_NAME`: Database name (default: flaskdb)

## Files

- `database_cloudinint.yaml`: Cloud-init cho database server
- `webserver_cloudinit.yaml`: Cloud-init cho web server (CẦN SỬA IP)
- `app.py`: Flask application với dynamic DB config
- `requirements.txt`: Python dependencies
