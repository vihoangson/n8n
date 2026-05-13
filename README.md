# n8n Self-Hosted trên VPS

Hướng dẫn cài đặt và sử dụng n8n automation platform tự host trên VPS với Docker Compose.

---

## Yêu cầu hệ thống

- VPS với Ubuntu 20.04+ / Debian 11+
- Docker & Docker Compose đã cài sẵn
- Tối thiểu 1GB RAM, 10GB disk
- Domain hoặc IP public (ví dụ: `app1.oop.vn`)

---

## Cài đặt nhanh

```bash
# 1. Clone hoặc tạo thư mục dự án
mkdir n8n-self-hosted && cd n8n-self-hosted

# 2. Tạo file docker-compose.yml (xem nội dung bên dưới)

# 3. Khởi động
docker compose up -d

# 4. Kiểm tra trạng thái
docker compose ps
docker compose logs -f n8n
```

Truy cập tại: **http://app1.oop.vn:5678**

---

## Cấu hình (docker-compose.yml)

```yaml
version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
      - TZ=Asia/Ho_Chi_Minh
      - N8N_HOST=app1.oop.vn
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://app1.oop.vn:5678/
      - N8N_SECURE_COOKIE=false
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=admin123
      - DB_TYPE=sqlite
      - DB_SQLITE_DATABASE=/home/node/.n8n/database.sqlite
      - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_VERSION_NOTIFICATIONS_ENABLED=false
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
    driver: local
```

> ⚠️ Đổi `N8N_BASIC_AUTH_USER` và `N8N_BASIC_AUTH_PASSWORD` trước khi deploy.

---

## Biến môi trường quan trọng

| Biến | Mô tả | Mặc định |
|------|-------|---------|
| `N8N_HOST` | Domain hoặc IP của server | `localhost` |
| `N8N_PORT` | Port lắng nghe | `5678` |
| `N8N_PROTOCOL` | `http` hoặc `https` | `http` |
| `WEBHOOK_URL` | URL base cho webhook | `http://localhost:5678/` |
| `N8N_SECURE_COOKIE` | Bật/tắt secure cookie (cần `false` khi dùng HTTP) | `true` |
| `N8N_BASIC_AUTH_ACTIVE` | Bật xác thực Basic Auth | `false` |
| `N8N_BASIC_AUTH_USER` | Tên đăng nhập | - |
| `N8N_BASIC_AUTH_PASSWORD` | Mật khẩu | - |
| `GENERIC_TIMEZONE` | Múi giờ | `UTC` |

---

## Workflow mẫu: Webhook → Telegram

Workflow cơ bản nhận dữ liệu qua HTTP và gửi thông báo Telegram.

### Các node

```
[Webhook] → [Set] → [Telegram]
```

### Cấu hình từng node

**1. Webhook**
- Method: `POST`
- Path: `contact-form`
- URL: `http://app1.oop.vn:5678/webhook/contact-form`

**2. Set**
- Field: `message`
- Value: `={{ $json.body.name }} vừa liên hệ: {{ $json.body.message }}`

**3. Telegram**
- Credential: Bot Token từ @BotFather
- Chat ID: ID Telegram của bạn
- Text: `={{ $json.message }}`

### Test

```bash
curl -X POST http://app1.oop.vn:5678/webhook/contact-form \
  -H "Content-Type: application/json" \
  -d '{"name": "Nguyen Van A", "message": "Xin chào!"}'
```

---

## Các lệnh Docker thường dùng

```bash
# Khởi động
docker compose up -d

# Dừng
docker compose down

# Khởi động lại
docker compose restart n8n

# Xem log realtime
docker compose logs -f n8n

# Cập nhật lên phiên bản mới nhất
docker compose pull
docker compose up -d

# Backup dữ liệu
docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/n8n-backup-$(date +%Y%m%d).tar.gz /data
```

---

## Nâng cấp lên HTTPS (khuyến nghị cho production)

Cài Nginx + Certbot để có SSL miễn phí:

```bash
# Cài Nginx và Certbot
apt install nginx certbot python3-certbot-nginx -y

# Lấy SSL certificate
certbot --nginx -d app1.oop.vn

# Cấu hình Nginx reverse proxy
# /etc/nginx/sites-available/n8n
server {
    listen 443 ssl;
    server_name app1.oop.vn;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Sau đó đổi trong `docker-compose.yml`:
```yaml
- N8N_PROTOCOL=https
- WEBHOOK_URL=https://app1.oop.vn/
- N8N_SECURE_COOKIE=true
```

---

## Ý tưởng workflow tiếp theo

| Workflow | Mô tả |
|----------|-------|
| Webhook → Google Sheets | Lưu data form vào spreadsheet tự động |
| Schedule → API → Telegram | Báo giá crypto / tỷ giá mỗi sáng |
| Gmail → Filter → Slack | Forward email quan trọng sang Slack |
| Webhook → OpenAI → Telegram | Chatbot đơn giản |

---

## Tài liệu tham khảo

- [n8n Docs](https://docs.n8n.io)
- [n8n Community](https://community.n8n.io)
- [Docker Hub - n8nio/n8n](https://hub.docker.com/r/n8nio/n8n)
