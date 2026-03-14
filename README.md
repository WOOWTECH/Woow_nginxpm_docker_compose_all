# Nginx Proxy Manager + PostgreSQL Docker Compose

Production-ready Docker Compose setup for **Nginx Proxy Manager** with **PostgreSQL 17** database backend.

適用於生產環境的 **Nginx Proxy Manager** Docker Compose 部署方案，搭配 **PostgreSQL 17** 資料庫後端。

---

## Table of Contents / 目錄

- [English](#english)
- [繁體中文](#繁體中文)

---

## English

### What is Nginx Proxy Manager?

[Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM) is a reverse proxy management system that provides:

- **Web-based Admin UI** — Manage proxy hosts, SSL certificates, and access lists through an intuitive web interface on port 81
- **Reverse Proxy** — Route incoming HTTP/HTTPS traffic to internal services by domain name
- **Free SSL Certificates** — Automatic Let's Encrypt SSL certificate provisioning and renewal
- **Access Lists** — Restrict access to proxied services by IP or HTTP authentication
- **Redirection Hosts** — Redirect domains to other URLs
- **404 Hosts** — Custom 404 pages for unused domains
- **Streams** — TCP/UDP port forwarding for non-HTTP services

### Architecture

```
┌──────────────────────────────────────────────┐
│              Docker Compose Stack             │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │     Nginx Proxy Manager (app)        │    │
│  │     Ports: 80, 443, 81              │    │
│  │     Image: jc21/nginx-proxy-manager  │    │
│  └──────────────┬───────────────────────┘    │
│                 │                             │
│                 │ DB_POSTGRES_HOST=db         │
│                 │                             │
│  ┌──────────────▼───────────────────────┐    │
│  │        PostgreSQL 17 (db)            │    │
│  │        Port: 5432 (internal)         │    │
│  │        Image: postgres:17            │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  Network: npm-network (bridge)               │
│  Volumes: npm-db-data, npm-app-data,         │
│           npm-letsencrypt                     │
└──────────────────────────────────────────────┘
```

### File Structure

```
.
├── docker-compose.yml    # Main orchestration file / 主要編排檔案
├── .env.example          # Configuration template / 環境變數範本
├── .env                  # Your configuration (git-ignored) / 你的設定檔（不納入版控）
├── .gitignore            # Git ignore rules / Git 忽略規則
└── README.md             # This documentation / 本說明文件
```

### Prerequisites

| Requirement | Minimum Version |
|-------------|----------------|
| Docker Engine | 20.10+ |
| Docker Compose | v2.0+ |
| RAM | 1 GB |
| Disk Space | 2 GB |

> **Note:** This setup is also compatible with **Podman** and `podman compose`.

### Quick Start

1. **Clone the repository:**
   ```bash
   git clone https://github.com/WOOWTECH/Woow_nginxpm_docker_compose_all.git
   cd Woow_nginxpm_docker_compose_all
   ```

2. **Configure environment variables:**
   ```bash
   cp .env.example .env
   # Edit .env and set a strong POSTGRES_PASSWORD
   nano .env
   ```

3. **Start the services:**
   ```bash
   docker compose up -d
   ```
   Or with Podman:
   ```bash
   podman compose up -d
   ```

4. **Access the Admin UI:**
   - URL: `http://<your-server-ip>:81`
   - Default Email: `admin@example.com`
   - Default Password: `changeme`
   - **Change the default credentials immediately after first login!**

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `POSTGRES_USER` | `npm` | PostgreSQL username |
| `POSTGRES_PASSWORD` | *(required)* | PostgreSQL password |
| `POSTGRES_DB` | `npm` | PostgreSQL database name |
| `NPM_HTTP_PORT` | `80` | Host port for HTTP traffic |
| `NPM_HTTPS_PORT` | `443` | Host port for HTTPS traffic |
| `NPM_ADMIN_PORT` | `81` | Host port for Admin UI |
| `TZ` | `Asia/Taipei` | Container timezone |

### Common Commands

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Start all services in background |
| `docker compose down` | Stop and remove containers |
| `docker compose logs -f` | Follow live logs |
| `docker compose logs -f app` | Follow NPM app logs only |
| `docker compose logs -f db` | Follow PostgreSQL logs only |
| `docker compose restart` | Restart all services |
| `docker compose pull` | Pull latest images |
| `docker compose up -d --pull always` | Update and restart |

### Data Persistence

Data is stored in three Docker named volumes:

| Volume | Purpose | Container Path |
|--------|---------|----------------|
| `npm-db-data` | PostgreSQL database | `/var/lib/postgresql/data` |
| `npm-app-data` | NPM configuration, proxy hosts, access lists | `/data` |
| `npm-letsencrypt` | SSL certificates from Let's Encrypt | `/etc/letsencrypt` |

### Backup & Restore

#### Database Backup

```bash
# Backup PostgreSQL database
docker compose exec db pg_dump -U npm npm > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore PostgreSQL database
docker compose exec -T db psql -U npm npm < backup_YYYYMMDD_HHMMSS.sql
```

#### Full Volume Backup

```bash
# Stop services first
docker compose down

# Backup all volumes
docker run --rm -v npm-db-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/npm-db-data_$(date +%Y%m%d).tar.gz -C /data .

docker run --rm -v npm-app-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/npm-app-data_$(date +%Y%m%d).tar.gz -C /data .

docker run --rm -v npm-letsencrypt:/data -v $(pwd):/backup alpine \
  tar czf /backup/npm-letsencrypt_$(date +%Y%m%d).tar.gz -C /data .

# Restart services
docker compose up -d
```

#### Volume Restore

```bash
docker compose down

docker run --rm -v npm-db-data:/data -v $(pwd):/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/npm-db-data_YYYYMMDD.tar.gz -C /data"

docker run --rm -v npm-app-data:/data -v $(pwd):/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/npm-app-data_YYYYMMDD.tar.gz -C /data"

docker run --rm -v npm-letsencrypt:/data -v $(pwd):/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/npm-letsencrypt_YYYYMMDD.tar.gz -C /data"

docker compose up -d
```

### Troubleshooting

| Issue | Diagnostic Command | Solution |
|-------|-------------------|----------|
| Container won't start | `docker compose logs app` | Check environment variables in `.env` |
| Database connection error | `docker compose logs db` | Verify `POSTGRES_PASSWORD` is set |
| Port 80/443 already in use | `sudo ss -tlnp \| grep ':80'` | Change `NPM_HTTP_PORT` / `NPM_HTTPS_PORT` in `.env` |
| Port 81 already in use | `sudo ss -tlnp \| grep ':81'` | Change `NPM_ADMIN_PORT` in `.env` |
| Cannot access admin UI | `docker compose ps` | Ensure containers are running and healthy |
| SSL certificate issues | `docker compose logs app` | Check DNS points to server, ports 80/443 open |

### How It Works — Setup Guide

1. **PostgreSQL starts first** — The `db` service initializes with the configured credentials and runs a health check (`pg_isready`).

2. **NPM waits for healthy DB** — The `app` service uses `depends_on` with `condition: service_healthy` to ensure the database is ready before starting.

3. **NPM connects to PostgreSQL** — Environment variables `DB_POSTGRES_HOST`, `DB_POSTGRES_PORT`, `DB_POSTGRES_USER`, `DB_POSTGRES_PASSWORD`, and `DB_POSTGRES_NAME` tell NPM how to connect.

4. **First-time initialization** — On first launch, NPM creates its database schema and a default admin user (`admin@example.com` / `changeme`).

5. **Proxy management** — Access the web UI on port 81 to configure reverse proxy hosts, SSL certificates, and access control rules.

---

## 繁體中文

### 什麼是 Nginx Proxy Manager？

[Nginx Proxy Manager](https://nginxproxymanager.com/)（NPM）是一套反向代理管理系統，提供以下功能：

- **網頁管理介面** — 透過直覺的網頁介面（連接埠 81）管理代理主機、SSL 憑證及存取控制清單
- **反向代理（Reverse Proxy）** — 依據網域名稱將 HTTP/HTTPS 流量路由到內部服務
- **免費 SSL 憑證** — 自動取得及續約 Let's Encrypt SSL 憑證
- **存取控制清單（Access Lists）** — 透過 IP 或 HTTP 驗證限制服務存取
- **重導向主機（Redirection Hosts）** — 將網域重導向至其他 URL
- **404 主機** — 為未使用的網域設定自訂 404 頁面
- **串流轉發（Streams）** — 支援非 HTTP 服務的 TCP/UDP 連接埠轉發

### 系統架構

```
┌──────────────────────────────────────────────┐
│            Docker Compose 服務堆疊            │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │   Nginx Proxy Manager（app 服務）     │    │
│  │   連接埠：80、443、81                 │    │
│  │   映像檔：jc21/nginx-proxy-manager   │    │
│  └──────────────┬───────────────────────┘    │
│                 │                             │
│                 │ DB_POSTGRES_HOST=db         │
│                 │                             │
│  ┌──────────────▼───────────────────────┐    │
│  │      PostgreSQL 17（db 服務）         │    │
│  │      連接埠：5432（僅內部）           │    │
│  │      映像檔：postgres:17             │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  網路：npm-network（bridge 模式）             │
│  磁碟區：npm-db-data、npm-app-data、         │
│          npm-letsencrypt                      │
└──────────────────────────────────────────────┘
```

### 檔案結構

```
.
├── docker-compose.yml    # 主要編排檔案
├── .env.example          # 環境變數範本
├── .env                  # 你的設定檔（不納入版控）
├── .gitignore            # Git 忽略規則
└── README.md             # 本說明文件
```

### 系統需求

| 需求項目 | 最低版本 |
|---------|---------|
| Docker Engine | 20.10+ |
| Docker Compose | v2.0+ |
| 記憶體（RAM） | 1 GB |
| 磁碟空間 | 2 GB |

> **備註：** 本方案亦相容 **Podman** 與 `podman compose`。

### 快速開始

1. **複製儲存庫：**
   ```bash
   git clone https://github.com/WOOWTECH/Woow_nginxpm_docker_compose_all.git
   cd Woow_nginxpm_docker_compose_all
   ```

2. **設定環境變數：**
   ```bash
   cp .env.example .env
   # 編輯 .env，設定一組強密碼給 POSTGRES_PASSWORD
   nano .env
   ```

3. **啟動服務：**
   ```bash
   docker compose up -d
   ```
   使用 Podman：
   ```bash
   podman compose up -d
   ```

4. **存取管理介面：**
   - 網址：`http://<你的伺服器 IP>:81`
   - 預設信箱：`admin@example.com`
   - 預設密碼：`changeme`
   - **首次登入後請立即更改預設帳號密碼！**

### 環境變數說明

| 變數名稱 | 預設值 | 說明 |
|----------|--------|------|
| `POSTGRES_USER` | `npm` | PostgreSQL 使用者名稱 |
| `POSTGRES_PASSWORD` | *（必填）* | PostgreSQL 密碼 |
| `POSTGRES_DB` | `npm` | PostgreSQL 資料庫名稱 |
| `NPM_HTTP_PORT` | `80` | 主機的 HTTP 連接埠 |
| `NPM_HTTPS_PORT` | `443` | 主機的 HTTPS 連接埠 |
| `NPM_ADMIN_PORT` | `81` | 主機的管理介面連接埠 |
| `TZ` | `Asia/Taipei` | 容器時區 |

### 常用指令

| 指令 | 說明 |
|------|------|
| `docker compose up -d` | 在背景啟動所有服務 |
| `docker compose down` | 停止並移除容器 |
| `docker compose logs -f` | 即時追蹤日誌 |
| `docker compose logs -f app` | 僅追蹤 NPM 應用程式日誌 |
| `docker compose logs -f db` | 僅追蹤 PostgreSQL 日誌 |
| `docker compose restart` | 重新啟動所有服務 |
| `docker compose pull` | 拉取最新映像檔 |
| `docker compose up -d --pull always` | 更新映像檔並重新啟動 |

### 資料持久化

資料儲存於三個 Docker 具名磁碟區：

| 磁碟區名稱 | 用途 | 容器內路徑 |
|-----------|------|-----------|
| `npm-db-data` | PostgreSQL 資料庫 | `/var/lib/postgresql/data` |
| `npm-app-data` | NPM 設定、代理主機、存取清單 | `/data` |
| `npm-letsencrypt` | Let's Encrypt SSL 憑證 | `/etc/letsencrypt` |

### 備份與還原

#### 資料庫備份

```bash
# 備份 PostgreSQL 資料庫
docker compose exec db pg_dump -U npm npm > backup_$(date +%Y%m%d_%H%M%S).sql

# 還原 PostgreSQL 資料庫
docker compose exec -T db psql -U npm npm < backup_YYYYMMDD_HHMMSS.sql
```

#### 完整磁碟區備份

```bash
# 先停止服務
docker compose down

# 備份所有磁碟區
docker run --rm -v npm-db-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/npm-db-data_$(date +%Y%m%d).tar.gz -C /data .

docker run --rm -v npm-app-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/npm-app-data_$(date +%Y%m%d).tar.gz -C /data .

docker run --rm -v npm-letsencrypt:/data -v $(pwd):/backup alpine \
  tar czf /backup/npm-letsencrypt_$(date +%Y%m%d).tar.gz -C /data .

# 重新啟動服務
docker compose up -d
```

#### 磁碟區還原

```bash
docker compose down

docker run --rm -v npm-db-data:/data -v $(pwd):/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/npm-db-data_YYYYMMDD.tar.gz -C /data"

docker run --rm -v npm-app-data:/data -v $(pwd):/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/npm-app-data_YYYYMMDD.tar.gz -C /data"

docker run --rm -v npm-letsencrypt:/data -v $(pwd):/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/npm-letsencrypt_YYYYMMDD.tar.gz -C /data"

docker compose up -d
```

### 疑難排解

| 問題 | 診斷指令 | 解決方式 |
|------|---------|---------|
| 容器無法啟動 | `docker compose logs app` | 檢查 `.env` 中的環境變數 |
| 資料庫連線錯誤 | `docker compose logs db` | 確認 `POSTGRES_PASSWORD` 已設定 |
| 連接埠 80/443 被占用 | `sudo ss -tlnp \| grep ':80'` | 修改 `.env` 中的 `NPM_HTTP_PORT` / `NPM_HTTPS_PORT` |
| 連接埠 81 被占用 | `sudo ss -tlnp \| grep ':81'` | 修改 `.env` 中的 `NPM_ADMIN_PORT` |
| 無法存取管理介面 | `docker compose ps` | 確認容器正在執行且狀態健康 |
| SSL 憑證問題 | `docker compose logs app` | 確認 DNS 指向伺服器，連接埠 80/443 已開啟 |

### 架設方式說明

1. **PostgreSQL 優先啟動** — `db` 服務以設定的帳號密碼初始化，並透過健康檢查（`pg_isready`）確認就緒。

2. **NPM 等待資料庫就緒** — `app` 服務使用 `depends_on` 搭配 `condition: service_healthy`，確保資料庫就緒後才啟動。

3. **NPM 連線至 PostgreSQL** — 透過環境變數 `DB_POSTGRES_HOST`、`DB_POSTGRES_PORT`、`DB_POSTGRES_USER`、`DB_POSTGRES_PASSWORD` 及 `DB_POSTGRES_NAME` 設定資料庫連線。

4. **首次初始化** — 首次啟動時，NPM 會自動建立資料庫結構及預設管理帳號（`admin@example.com` / `changeme`）。

5. **代理管理** — 透過連接埠 81 的網頁介面設定反向代理主機、SSL 憑證及存取控制規則。

---

### AI Deployment Guide / AI 部署指南

For AI-assisted deployment, here are the structured specifications:

```yaml
# Service Expectations
services:
  app:
    expected_state: running
    health_indicator: HTTP 200 on port 81
    startup_time: ~30 seconds
    depends: db (healthy)
  db:
    expected_state: running
    health_indicator: pg_isready returns 0
    startup_time: ~10 seconds

# Required Environment Variables
required:
  - POSTGRES_PASSWORD  # Must be set, no default

# Network Architecture
network:
  type: bridge
  name: npm-network
  internal_communication: app ↔ db via hostname "db"
  external_ports: [80, 443, 81]
```
