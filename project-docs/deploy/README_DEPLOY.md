# Deploy Production Public Link

Mục tiêu: đưa dashboard lên một server riêng để ai có link đều dùng được, không phụ thuộc máy local và không ảnh hưởng `Start Local Dashboard.cmd` hoặc `Public Dashboard.cmd`.

## Kiến Trúc

```text
User -> https://DASHBOARD_DOMAIN
        -> Caddy container
           -> static frontend build
           -> /api/v1/* reverse proxy nội bộ tới backend:8001
              -> FastAPI backend container
```

Backend không mở port trực tiếp ra internet. Người dùng chỉ thấy dashboard và gọi API qua cùng domain.

## Cần Chuẩn Bị

- Một VPS Ubuntu có Docker và Docker Compose plugin.
- Một domain/subdomain trỏ DNS `A record` về IP của VPS.
- Port `80` và `443` mở trên firewall.

Gợi ý cấu hình VPS:

- Dùng cơ bản: 2 CPU, 4 GB RAM.
- Nếu chạy nhiều model deep learning: 4 CPU, 8-16 GB RAM trở lên.

## Cài Docker Trên VPS

Làm theo tài liệu chính thức của Docker cho Ubuntu. Sau khi cài, kiểm tra:

```bash
docker --version
docker compose version
```

## Đưa Project Lên VPS

Nên để repo/private source trên VPS tại:

```bash
/opt/vn-finance-dashboard
```

Ví dụ:

```bash
sudo mkdir -p /opt/vn-finance-dashboard
sudo chown -R $USER:$USER /opt/vn-finance-dashboard
```

Sau đó copy project lên thư mục này bằng `scp`, Git private repo, hoặc công cụ deploy riêng của bạn.

Không đưa project lên public GitHub nếu bạn muốn giữ kín source code.

## Cấu Hình Domain

Tạo file env production:

```bash
cd /opt/vn-finance-dashboard/deploy
cp .env.production.example .env
nano .env
```

Sửa:

```bash
DASHBOARD_DOMAIN=dashboard.example.com
```

Nếu chưa có domain và chỉ test bằng IP:

```bash
DASHBOARD_DOMAIN=:80
```

## Chạy Production

Từ thư mục `deploy`:

```bash
docker compose --env-file .env -f docker-compose.prod.yml up -d --build
```

Kiểm tra:

```bash
docker compose --env-file .env -f docker-compose.prod.yml ps
docker compose --env-file .env -f docker-compose.prod.yml logs -f --tail=100
```

Mở:

```text
https://dashboard.example.com
```

Nếu dùng IP test:

```text
http://SERVER_IP
```

## Cập Nhật Phiên Bản Mới

Sau khi copy code mới lên VPS:

```bash
cd /opt/vn-finance-dashboard/deploy
docker compose --env-file .env -f docker-compose.prod.yml up -d --build
```

## Bảo Mật

- Chỉ mở port `80` và `443` ra internet.
- Không mở port `8001`; backend chỉ nằm trong Docker network.
- Không commit file `deploy/.env`.
- Không đưa repo lên public.
- Nếu VPS có firewall:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

## Không Ảnh Hưởng Local/Public Hiện Tại

Bộ production này dùng các file riêng:

```text
deploy/docker-compose.prod.yml
deploy/Caddyfile
backend/Dockerfile.prod
frontend/Dockerfile.prod
```

Nó không sửa `scripts/dev.ps1`, `scripts/public.ps1`, `Start Local Dashboard.cmd` hay `Public Dashboard.cmd`.
