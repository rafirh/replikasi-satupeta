# Dokumen Requirement untuk Replikasi Portal Satu Peta

## Daftar Isi
1. [Overview Sistem](#overview-sistem)
2. [Requirements Hardware & Software](#requirements-hardware--software)
3. [Database Setup](#database-setup)
4. [Object Storage (MinIO/S3)](#object-storage-minios3)
5. [Backend Setup](#backend-setup)
6. [Frontend Setup](#frontend-setup)
7. [Environment Variables](#environment-variables)
8. [Deployment dengan Docker](#deployment-dengan-docker)
9. [Monitoring & Maintenance](#monitoring--maintenance)

---

## Overview Sistem

Portal Satu Peta adalah aplikasi web untuk manajemen data peta yang terdiri dari:
- **Backend**: FastAPI (Python 3.13) dengan PostgreSQL database
- **Frontend**: Next.js 15 (React 19) dengan TypeScript
- **Storage**: MinIO/S3 untuk penyimpanan file
- **Map Services**: Integrasi dengan Cesium dan Leaflet untuk visualisasi peta

### Arsitektur Aplikasi
```
┌─────────────┐      ┌──────────────┐      ┌──────────────┐
│   Frontend  │────▶ │   Backend    │────▶ │  PostgreSQL  │
│  (Next.js)  │      │  (FastAPI)   │      │   Database   │
└─────────────┘      └──────────────┘      └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │    MinIO     │
                     │  (S3 Storage)│
                     └──────────────┘
```

---

## Requirements Hardware & Software

### Hardware Requirements (Minimum)

#### Development Environment
- **CPU**: 2 cores
- **RAM**: 4 GB
- **Storage**: 20 GB SSD
- **Network**: Koneksi internet stabil

#### Production Environment
- **CPU**: 4+ cores
- **RAM**: 8+ GB (16 GB recommended)
- **Storage**: 100+ GB SSD (tergantung volume data peta)
- **Network**: Bandwidth tinggi untuk transfer data geospasial

### Software Requirements

#### Operating System
- **Linux**: Ubuntu 20.04 LTS / 22.04 LTS (recommended)
- **macOS**: 11.x atau lebih baru
- **Windows**: Windows 10/11 dengan WSL2

#### Required Software

1. **Python**
   - Version: 3.13.x
   - Package manager: Poetry 1.7.1+

2. **Node.js & Package Manager**
   - Node.js: v20.x LTS
   - pnpm: latest (via corepack)

3. **Database**
   - PostgreSQL: 14.x atau lebih baru
   - PostgreSQL Extensions:
     - `postgis` (untuk data geospasial)
     - `uuid-ossp` (untuk UUID generation)

4. **Object Storage**
   - MinIO: latest stable version
   - Atau AWS S3 / compatible S3 storage

5. **Docker (Optional tapi Recommended)**
   - Docker Engine: 20.10.x atau lebih baru
   - Docker Compose: 2.x

6. **Version Control**
   - Git: 2.x

7. **Web Server / Reverse Proxy (Production)**
   - Nginx: 1.18+ atau
   - Apache: 2.4+

---

## Database Setup

### 1. Install PostgreSQL

#### Ubuntu/Debian
```bash
# Install PostgreSQL dan PostGIS
sudo apt update
sudo apt install postgresql postgresql-contrib postgis

# Verifikasi instalasi
psql --version
```

#### macOS (dengan Homebrew)
```bash
brew install postgresql@14 postgis
brew services start postgresql@14
```

#### Docker (Alternative)
```bash
docker run -d \
  --name satupeta-postgres \
  -e POSTGRES_DB=satupeta \
  -e POSTGRES_USER=satupeta_user \
  -e POSTGRES_PASSWORD=strong_password_here \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgis/postgis:14-3.3
```

### 2. Konfigurasi Database

```bash
# Login ke PostgreSQL
sudo -u postgres psql

# Atau jika menggunakan user lain
psql -U postgres
```

```sql
-- Buat database
CREATE DATABASE satupeta;

-- Buat user
CREATE USER satupeta_user WITH ENCRYPTED PASSWORD 'strong_password_here';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE satupeta TO satupeta_user;

-- Connect ke database
\c satupeta

-- Enable ekstensi yang diperlukan
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS postgis;

-- Grant privileges untuk schema
GRANT ALL ON SCHEMA public TO satupeta_user;

-- Verifikasi ekstensi
\dx
```

### 3. Connection String

Format connection string untuk backend:
```
postgresql://[username]:[password]@[host]:[port]/[database]
```

Contoh:
```
postgresql://satupeta_user:strong_password_here@localhost:5432/satupeta
```

### 4. Database Configuration untuk Production

Edit file `/etc/postgresql/14/main/postgresql.conf`:

```conf
# Connection Settings
max_connections = 100
shared_buffers = 256MB
effective_cache_size = 1GB
maintenance_work_mem = 64MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2621kB
min_wal_size = 1GB
max_wal_size = 4GB

# Locale settings untuk Indonesia
lc_messages = 'id_ID.UTF-8'
lc_monetary = 'id_ID.UTF-8'
lc_numeric = 'id_ID.UTF-8'
lc_time = 'id_ID.UTF-8'
timezone = 'Asia/Jakarta'
```

Edit file `/etc/postgresql/14/main/pg_hba.conf` untuk mengatur akses:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     peer
host    satupeta        satupeta_user   0.0.0.0/0              md5
host    satupeta        satupeta_user   ::0/0                  md5
```

Restart PostgreSQL:
```bash
sudo systemctl restart postgresql
```

---

## Object Storage (MinIO/S3)

### 1. Install MinIO

#### Docker (Recommended)
```bash
docker run -d \
  --name satupeta-minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin123 \
  -v minio_data:/data \
  minio/minio server /data --console-address ":9001"
```

#### Binary Installation (Linux)
```bash
# Download MinIO
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/

# Buat direktori data
sudo mkdir -p /data/minio

# Buat service file
sudo nano /etc/systemd/system/minio.service
```

Content file service:
```ini
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local/

User=minio-user
Group=minio-user

Environment="MINIO_ROOT_USER=minioadmin"
Environment="MINIO_ROOT_PASSWORD=minioadmin123"

ExecStart=/usr/local/bin/minio server /data/minio --console-address ":9001"

Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# Buat user untuk MinIO
sudo useradd -r minio-user -s /sbin/nologin
sudo chown -R minio-user:minio-user /data/minio

# Start service
sudo systemctl daemon-reload
sudo systemctl enable minio
sudo systemctl start minio
```

### 2. Konfigurasi MinIO

1. Akses MinIO Console di `http://localhost:9001`
2. Login dengan credentials:
   - Username: `minioadmin`
   - Password: `minioadmin123`

3. Buat bucket baru:
   - Bucket name: `satu-peta`
   - Region: (kosongkan atau `us-east-1`)
   - Versioning: Enabled (optional)

4. Buat Access Key untuk aplikasi:
   - Navigate ke: Identity → Service Accounts
   - Create New Service Account
   - Simpan Access Key dan Secret Key

### 3. MinIO Configuration

Environment variables yang dibutuhkan:
```env
MINIO_ENDPOINT_URL=http://localhost:9000
MINIO_ROOT_USER=your_access_key
MINIO_ROOT_PASSWORD=your_secret_key
MINIO_SECURE=false
MINIO_BUCKET_NAME=satu-peta
MINIO_REGION=
```

**Note**: Untuk production, gunakan HTTPS dan set `MINIO_SECURE=true`

---

## Backend Setup

### 1. Clone Repository

```bash
cd /path/to/your/projects
git clone <repository-url> satupeta
cd satupeta/portal-satu-peta-backend-main
```

### 2. Install Poetry

```bash
# Linux/macOS
curl -sSL https://install.python-poetry.org | python3 -

# Atau via pip
pip install poetry==1.7.1
```

### 3. Install Dependencies

```bash
# Install semua dependencies
poetry install

# Atau install tanpa dev dependencies (production)
poetry install --without dev
```

### 4. Environment Configuration

Buat file `.env` dari template:

```bash
cp .env.example .env
```

Edit file `.env`:

```env
# Application Settings
PROJECT_NAME=Satu Peta
VERSION=0.1.0
DESCRIPTION=Satu Peta API
DEBUG=false

# Server Settings
HOST=0.0.0.0
PORT=5000
WORKERS=4
LOG_LEVEL=info
LOOP=uvloop
HTTP=httptools
LIMIT_CONCURRENCY=100
BACKLOG=2048

# Database Settings
DATABASE_URL=postgresql://satupeta_user:strong_password_here@localhost:5432/satupeta

# Security Settings
SECRET_KEY=your-super-secret-key-min-32-characters-long-here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# CORS Settings
ALLOWED_ORIGINS=["http://localhost:4000","https://yourdomain.com"]

# MinIO/S3 Settings
MINIO_ENDPOINT_URL=http://localhost:9000
MINIO_ROOT_USER=your_access_key
MINIO_ROOT_PASSWORD=your_secret_key
MINIO_SECURE=false
MINIO_BUCKET_NAME=satu-peta
MINIO_REGION=

# Upload Settings
MAX_UPLOAD_SIZE=104857600
ALLOWED_EXTENSIONS=["jpg","jpeg","png","pdf","doc","docx","xls","xlsx","txt","csv","zip","rar","json"]

# Timezone
TIMEZONE=Asia/Jakarta
```

**Generate SECRET_KEY:**
```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

### 5. Database Migration

```bash
# Jalankan migrasi
poetry run alembic upgrade head

# Atau menggunakan script custom
poetry run python migrations/scripts.py
```

### 6. Run Backend (Development)

```bash
# Menggunakan script run.py
poetry run python run.py

# Atau langsung dengan uvicorn
poetry run uvicorn app.main:app --host 0.0.0.0 --port 5000 --reload
```

Backend akan berjalan di: `http://localhost:5000`

API Documentation:
- Swagger UI: `http://localhost:5000/api/docs`
- ReDoc: `http://localhost:5000/api/redoc`

### 7. Testing

```bash
# Jalankan test
poetry run pytest

# Dengan coverage report
poetry run pytest --cov=app --cov-report=html
```

---

## Frontend Setup

### 1. Navigate ke Frontend Directory

```bash
cd /path/to/your/projects/satupeta/satupeta-frontend-main
```

### 2. Install pnpm

```bash
# Enable corepack (sudah termasuk di Node.js 16+)
corepack enable

# Prepare pnpm
corepack prepare pnpm@latest --activate

# Verifikasi instalasi
pnpm --version
```

### 3. Install Dependencies

```bash
pnpm install
```

### 4. Environment Configuration

Buat file `.env.local`:

```bash
touch .env.local
```

Edit `.env.local`:

```env
# Backend API URL
NEXT_PUBLIC_API=http://localhost:5000/api

# NextAuth Configuration
AUTH_SECRET=your-nextauth-secret-key-here
AUTH_URL=http://localhost:4000

# Database URL (jika ada direct database access dari frontend)
DATABASE_URL=postgresql://satupeta_user:strong_password_here@localhost:5432/satupeta

# Optional: Analytics, monitoring, dll
# NEXT_PUBLIC_GA_ID=
# NEXT_PUBLIC_SENTRY_DSN=
```

**Generate AUTH_SECRET:**
```bash
openssl rand -base64 32
```

### 5. Run Frontend (Development)

```bash
pnpm dev
```

Frontend akan berjalan di: `http://localhost:4000`

### 6. Build untuk Production

```bash
# Build aplikasi
pnpm build

# Test production build locally
pnpm start
```

---

## Environment Variables

### Backend Environment Variables (`.env`)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PROJECT_NAME` | No | Satu Peta | Nama aplikasi |
| `VERSION` | No | 0.1.0 | Versi aplikasi |
| `DEBUG` | No | false | Mode debug |
| `HOST` | No | 0.0.0.0 | Host server |
| `PORT` | No | 5000 | Port server |
| `DATABASE_URL` | **Yes** | - | PostgreSQL connection string |
| `SECRET_KEY` | **Yes** | - | Secret key untuk JWT (min 32 char) |
| `ALGORITHM` | No | HS256 | Algorithm untuk JWT |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | No | 30 | Expiry access token (menit) |
| `REFRESH_TOKEN_EXPIRE_DAYS` | No | 7 | Expiry refresh token (hari) |
| `ALLOWED_ORIGINS` | No | ["*"] | CORS allowed origins |
| `MINIO_ENDPOINT_URL` | **Yes** | - | MinIO/S3 endpoint |
| `MINIO_ROOT_USER` | **Yes** | - | MinIO access key |
| `MINIO_ROOT_PASSWORD` | **Yes** | - | MinIO secret key |
| `MINIO_SECURE` | No | false | Use HTTPS untuk MinIO |
| `MINIO_BUCKET_NAME` | No | satu-peta | Nama bucket |
| `TIMEZONE` | No | Asia/Jakarta | Timezone aplikasi |

### Frontend Environment Variables (`.env.local`)

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NEXT_PUBLIC_API` | **Yes** | - | Backend API base URL |
| `AUTH_SECRET` | **Yes** | - | NextAuth secret key |
| `AUTH_URL` | **Yes** | - | NextAuth base URL |
| `DATABASE_URL` | No | - | Database URL (jika perlu) |

---

## Deployment dengan Docker

### 1. Docker Compose Setup (Full Stack)

Buat file `docker-compose.yml` di root project:

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgis/postgis:14-3.3
    container_name: satupeta-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: satupeta
      POSTGRES_USER: satupeta_user
      POSTGRES_PASSWORD: strong_password_here
      POSTGRES_INITDB_ARGS: "-E UTF8 --locale=id_ID.UTF-8"
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - satupeta-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U satupeta_user -d satupeta"]
      interval: 10s
      timeout: 5s
      retries: 5

  # MinIO Object Storage
  minio:
    image: minio/minio:latest
    container_name: satupeta-minio
    restart: unless-stopped
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    networks:
      - satupeta-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # Backend API
  backend:
    build:
      context: ./portal-satu-peta-backend-main
      dockerfile: Dockerfile
    container_name: satupeta-backend
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      DATABASE_URL: postgresql://satupeta_user:strong_password_here@postgres:5432/satupeta
      MINIO_ENDPOINT_URL: http://minio:9000
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
      MINIO_SECURE: "false"
      SECRET_KEY: your-super-secret-key-here
      ALLOWED_ORIGINS: '["http://localhost:4000","http://localhost"]'
      DEBUG: "false"
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_healthy
    networks:
      - satupeta-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/api/docs"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Frontend
  frontend:
    build:
      context: ./satupeta-frontend-main
      dockerfile: Dockerfile
      args:
        NEXT_PUBLIC_API: http://localhost:5000/api
    container_name: satupeta-frontend
    restart: unless-stopped
    ports:
      - "4000:4000"
    environment:
      NEXT_PUBLIC_API: http://backend:5000/api
      AUTH_SECRET: your-nextauth-secret-here
      AUTH_URL: http://localhost:4000
    depends_on:
      - backend
    networks:
      - satupeta-network

networks:
  satupeta-network:
    driver: bridge

volumes:
  postgres_data:
  minio_data:
```

### 2. Deploy dengan Docker Compose

```bash
# Build dan start semua services
docker-compose up -d --build

# Check logs
docker-compose logs -f

# Check status
docker-compose ps

# Stop semua services
docker-compose down

# Stop dan hapus volumes (HATI-HATI: akan menghapus data!)
docker-compose down -v
```

### 3. Backend-only dengan Docker

```bash
cd portal-satu-peta-backend-main

# Build image
docker build -t satupeta-backend:latest .

# Run container
docker run -d \
  --name satupeta-backend \
  -p 5000:5000 \
  --env-file .env \
  satupeta-backend:latest
```

### 4. Frontend-only dengan Docker

```bash
cd satupeta-frontend-main

# Build image
docker build \
  --build-arg NEXT_PUBLIC_API=http://localhost:5000/api \
  -t satupeta-frontend:latest .

# Run container
docker run -d \
  --name satupeta-frontend \
  -p 4000:4000 \
  -e NEXT_PUBLIC_API=http://localhost:5000/api \
  -e AUTH_SECRET=your-secret-here \
  satupeta-frontend:latest
```

---

## Production Deployment

### 1. Nginx Reverse Proxy Configuration

Install Nginx:
```bash
sudo apt install nginx
```

Buat konfigurasi file `/etc/nginx/sites-available/satupeta`:

```nginx
# HTTP Redirect to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS - Frontend
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss;

    # Frontend
    location / {
        proxy_pass http://localhost:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Backend API
    location /api/ {
        proxy_pass http://localhost:5000/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Increase timeouts for large file uploads
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
        send_timeout 600;
        client_max_body_size 100M;
    }

    # MinIO/S3 Storage (optional, jika perlu akses langsung)
    location /storage/ {
        proxy_pass http://localhost:9000/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 100M;
    }
}
```

Enable site dan reload Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/satupeta /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 2. SSL Certificate dengan Let's Encrypt

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx

# Dapatkan certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Auto-renewal sudah disetup otomatis, test dengan:
sudo certbot renew --dry-run
```

### 3. Systemd Service untuk Backend

Buat file `/etc/systemd/system/satupeta-backend.service`:

```ini
[Unit]
Description=Satu Peta Backend API
After=network.target postgresql.service

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/opt/satupeta/portal-satu-peta-backend-main
Environment="PATH=/opt/satupeta/.venv/bin"
EnvironmentFile=/opt/satupeta/portal-satu-peta-backend-main/.env
ExecStart=/opt/satupeta/.venv/bin/python run.py
Restart=always
RestartSec=10

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/satupeta

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable satupeta-backend
sudo systemctl start satupeta-backend
sudo systemctl status satupeta-backend
```

### 4. PM2 untuk Frontend (Alternative)

```bash
# Install PM2 globally
npm install -g pm2

# Navigate ke frontend directory
cd /opt/satupeta/satupeta-frontend-main

# Start dengan PM2
pm2 start npm --name "satupeta-frontend" -- start

# Save PM2 configuration
pm2 save

# Setup PM2 startup
pm2 startup
```

---

## Monitoring & Maintenance

### 1. Log Locations

#### Backend Logs
```bash
# Jika menggunakan systemd
sudo journalctl -u satupeta-backend -f

# Jika menggunakan Docker
docker logs -f satupeta-backend
```

#### Frontend Logs
```bash
# PM2 logs
pm2 logs satupeta-frontend

# Docker logs
docker logs -f satupeta-frontend
```

#### Nginx Logs
```bash
# Access logs
sudo tail -f /var/log/nginx/access.log

# Error logs
sudo tail -f /var/log/nginx/error.log
```

### 2. Database Backup

Script backup otomatis (`/opt/scripts/backup-satupeta.sh`):

```bash
#!/bin/bash

BACKUP_DIR="/opt/backups/satupeta"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="satupeta"
DB_USER="satupeta_user"
DB_PASSWORD="strong_password_here"

# Create backup directory
mkdir -p $BACKUP_DIR

# Database backup
PGPASSWORD=$DB_PASSWORD pg_dump -U $DB_USER $DB_NAME | gzip > $BACKUP_DIR/db_backup_$DATE.sql.gz

# MinIO data backup (optional)
# mc mirror /data/minio $BACKUP_DIR/minio_$DATE/

# Keep only last 7 days backups
find $BACKUP_DIR -name "db_backup_*.sql.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
```

Setup cron job untuk backup otomatis:
```bash
sudo crontab -e

# Tambahkan line berikut untuk backup harian jam 2 pagi
0 2 * * * /opt/scripts/backup-satupeta.sh >> /var/log/satupeta-backup.log 2>&1
```

### 3. Database Restore

```bash
# Extract backup
gunzip db_backup_YYYYMMDD_HHMMSS.sql.gz

# Restore database
psql -U satupeta_user -d satupeta < db_backup_YYYYMMDD_HHMMSS.sql
```

### 4. Health Check Script

Buat file `/opt/scripts/healthcheck-satupeta.sh`:

```bash
#!/bin/bash

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m'

echo "=== Satu Peta Health Check ==="

# Check Backend
echo -n "Backend API: "
if curl -s -f http://localhost:5000/api/docs > /dev/null; then
    echo -e "${GREEN}OK${NC}"
else
    echo -e "${RED}FAIL${NC}"
fi

# Check Frontend
echo -n "Frontend: "
if curl -s -f http://localhost:4000 > /dev/null; then
    echo -e "${GREEN}OK${NC}"
else
    echo -e "${RED}FAIL${NC}"
fi

# Check Database
echo -n "PostgreSQL: "
if pg_isready -h localhost -p 5432 > /dev/null 2>&1; then
    echo -e "${GREEN}OK${NC}"
else
    echo -e "${RED}FAIL${NC}"
fi

# Check MinIO
echo -n "MinIO: "
if curl -s -f http://localhost:9000/minio/health/live > /dev/null; then
    echo -e "${GREEN}OK${NC}"
else
    echo -e "${RED}FAIL${NC}"
fi

# Disk Space
echo ""
echo "=== Disk Usage ==="
df -h | grep -E '(Filesystem|/$|/data)'

# Memory Usage
echo ""
echo "=== Memory Usage ==="
free -h

echo ""
echo "=== Docker Containers (if using) ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" 2>/dev/null || echo "Not using Docker"
```

Buat executable:
```bash
chmod +x /opt/scripts/healthcheck-satupeta.sh
```

### 5. Update Procedure

#### Backend Update
```bash
cd /opt/satupeta/portal-satu-peta-backend-main

# Pull latest changes
git pull origin main

# Install new dependencies
poetry install

# Run migrations
poetry run alembic upgrade head

# Restart service
sudo systemctl restart satupeta-backend
```

#### Frontend Update
```bash
cd /opt/satupeta/satupeta-frontend-main

# Pull latest changes
git pull origin main

# Install new dependencies
pnpm install

# Build
pnpm build

# Restart
pm2 restart satupeta-frontend
```

---

## Troubleshooting

### Common Issues

#### 1. Backend tidak bisa connect ke Database

**Solusi:**
- Pastikan PostgreSQL berjalan: `sudo systemctl status postgresql`
- Check connection string di `.env`
- Test koneksi manual: `psql -U satupeta_user -d satupeta -h localhost`
- Check firewall: `sudo ufw status`

#### 2. Frontend tidak bisa hit Backend API

**Solusi:**
- Check CORS settings di backend `ALLOWED_ORIGINS`
- Pastikan `NEXT_PUBLIC_API` di frontend sudah benar
- Check network connectivity
- Look at browser console untuk error messages

#### 3. File upload error ke MinIO

**Solusi:**
- Check MinIO service status
- Verify credentials di backend `.env`
- Check bucket exists dan accessible
- Verify `MAX_UPLOAD_SIZE` setting

#### 4. Database migration error

**Solusi:**
```bash
# Check current migration version
poetry run alembic current

# Check migration history
poetry run alembic history

# Downgrade dan upgrade lagi
poetry run alembic downgrade -1
poetry run alembic upgrade head
```

---

## Security Checklist

### Production Security

- [ ] Ganti semua default passwords (PostgreSQL, MinIO, dll)
- [ ] Generate strong `SECRET_KEY` dan `AUTH_SECRET`
- [ ] Setup SSL/TLS certificates
- [ ] Konfigurasi firewall (ufw/iptables)
- [ ] Set `DEBUG=false` di production
- [ ] Limit CORS origins ke domain spesifik
- [ ] Setup rate limiting di Nginx
- [ ] Enable database connection SSL
- [ ] Implement backup strategy
- [ ] Setup monitoring dan alerting
- [ ] Keep dependencies up to date
- [ ] Regular security audits

### Firewall Configuration

```bash
# Allow SSH
sudo ufw allow 22/tcp

# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Block direct access ke backend (akses lewat Nginx saja)
# sudo ufw deny 5000/tcp

# Enable firewall
sudo ufw enable
sudo ufw status
```

---

## Appendix

### A. Port Reference

| Service | Default Port | Purpose |
|---------|-------------|---------|
| Frontend | 4000 | Next.js application |
| Backend | 5000 | FastAPI application |
| PostgreSQL | 5432 | Database |
| MinIO API | 9000 | Object storage API |
| MinIO Console | 9001 | MinIO web UI |
| Nginx | 80, 443 | Reverse proxy |

### B. Useful Commands

```bash
# Check port usage
sudo netstat -tlnp | grep :5000

# Check process
ps aux | grep python
ps aux | grep node

# Disk usage
du -sh /opt/satupeta/*
df -h

# Memory usage
free -m
top

# Docker cleanup
docker system prune -a --volumes

# Poetry commands
poetry show                    # List installed packages
poetry update                  # Update dependencies
poetry export -f requirements.txt --output requirements.txt
```

### C. Contact & Support

Untuk bantuan lebih lanjut:
- GitHub Issues: [repository-url]/issues
- Email: trikintech@gmail.com

---

**Document Version**: 1.0  
**Last Updated**: November 26, 2025  
**Maintained By**: Development Team
