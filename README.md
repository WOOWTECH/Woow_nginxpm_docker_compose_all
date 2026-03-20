# Nginx Proxy Manager - K3s/Kubernetes 部署指南

[English](#english) | [中文](#中文)

---

## English

### Overview

Easy-to-use reverse proxy with automatic SSL certificate management (Let's Encrypt) and a web-based GUI. Nginx Proxy Manager simplifies the process of exposing internal services to the internet with SSL termination, custom domains, access lists, and redirect rules. It eliminates the need to manually edit Nginx configuration files by providing a clean web interface for managing proxy hosts.

> **GitHub Repo (Podman/Docker):** [Woow_nginxpm_docker_compose_all](https://github.com/WOOWTECH/Woow_nginxpm_docker_compose_all)

### Architecture

```
                         ┌──────────────────────────────────────────────────────┐
                         │                  K3s / Kubernetes                    │
                         │                                                      │
                         │  ┌──────────────────────────────────────────────┐    │
  ┌───────────┐          │  │           Namespace: nginxpm                  │    │
  │  Browser   │          │  │                                              │    │
  │  (Admin)   │  :30081  │  │  ┌────────────────┐    ┌─────────────────┐  │    │
  │           ├──────────►│  │  │ Service        │    │  Deployment     │  │    │
  └───────────┘  NodePort │  │  │ nginxpm :81    ├───►│  nginxpm        │  │    │
                         │  │  └────────────────┘    │ (jc21/nginx-    │  │    │
  ┌───────────┐          │  │                         │  proxy-manager) │  │    │
  │  HTTP      │  :30080  │  │  ┌────────────────┐    │                 │  │    │
  │  Traffic   ├──────────►│  │  │ Service        │    │  :80  (HTTP)    │  │    │
  └───────────┘  NodePort │  │  │ nginxpm :80    ├───►│  :443 (HTTPS)   │  │    │
                         │  │  └────────────────┘    │  :81  (Admin)    │  │    │
  ┌───────────┐          │  │                         │                 │  │    │
  │  HTTPS     │  :30443  │  │  ┌────────────────┐    │ [PVC: 5Gi data] │  │    │
  │  Traffic   ├──────────►│  │  │ Service        │    │ [PVC: 1Gi certs]│  │    │
  └───────────┘  NodePort │  │  │ nginxpm :443   ├───►│                 │  │    │
                         │  │  └────────────────┘    └────────┬────────┘  │    │
                         │  │                                  │           │    │
                         │  └──────────────────────────────────┼───────────┘    │
                         │                                      │               │
                         │          Proxied upstream services    ▼               │
                         │  ┌───────────────────────────────────────────────┐   │
                         │  │  nextcloud.nextcloud.svc:80                   │   │
                         │  │  odoo.odoo.svc:8069                           │   │
                         │  │  n8n.n8n.svc:5678                             │   │
                         │  │  immich-server.immich.svc:2283                │   │
                         │  │  homeassistant.homeassistant.svc:8123         │   │
                         │  │  ... (any cluster service)                    │   │
                         │  └───────────────────────────────────────────────┘   │
                         └──────────────────────────────────────────────────────┘

  Port Mappings:
    External :30080  ──►  Service :80   ──►  Pod nginxpm :80   (HTTP proxy)
    External :30443  ──►  Service :443  ──►  Pod nginxpm :443  (HTTPS proxy)
    External :30081  ──►  Service :81   ──►  Pod nginxpm :81   (Admin panel)
```

### Features

- Web-based GUI for managing reverse proxy configurations
- Automatic SSL certificate provisioning via Let's Encrypt
- SSL termination with HTTP/2 support
- Access lists for IP-based restrictions
- Custom Nginx configuration support
- Redirect and stream rules
- Websocket proxying support (required for Home Assistant, n8n, etc.)
- Built-in SQLite database (no external database required)

### Quick Start

```bash
# 1. Deploy Nginx Proxy Manager
kubectl apply -k k8s-manifests/nginxpm/

# 2. Verify pods are running
kubectl -n nginxpm get pods

# 3. Access the admin panel
echo "http://<node-ip>:30081"
```

### Configuration

#### No Environment Variables or Secrets Required

Nginx Proxy Manager is configured entirely through its web interface. No ConfigMap or Secret files are used in this deployment.

#### Default Admin Credentials

| Setting | Value |
|---------|-------|
| Email | `admin@example.com` |
| Password | `changeme` |

**Change these immediately after first login.**

### Accessing the Service

| Endpoint | URL | Protocol |
|----------|-----|----------|
| Admin Panel | `http://<node-ip>:30081` | HTTP (NodePort) |
| HTTP Proxy | `http://<node-ip>:30080` | HTTP (NodePort) |
| HTTPS Proxy | `https://<node-ip>:30443` | HTTPS (NodePort) |
| Internal Admin | `http://nginxpm.nginxpm.svc.cluster.local:81` | HTTP |
| Internal HTTP | `http://nginxpm.nginxpm.svc.cluster.local:80` | HTTP |
| Internal HTTPS | `https://nginxpm.nginxpm.svc.cluster.local:443` | HTTPS |

#### Port Summary

| Port | NodePort | Purpose |
|------|----------|---------|
| 80 | 30080 | HTTP traffic (proxy) |
| 443 | 30443 | HTTPS traffic (proxy with SSL) |
| 81 | 30081 | Admin panel web UI |

### Data Persistence

| PVC Name | Mount Path | Size | Purpose |
|----------|------------|------|---------|
| `nginxpm-data` | `/data` | 5Gi | Proxy configurations, access lists, custom locations, SQLite database |
| `nginxpm-letsencrypt` | `/etc/letsencrypt` | 1Gi | Let's Encrypt SSL certificates and ACME account data |

All PVCs use the `local-path` storage class (k3s default).

### Setting Up Proxy Hosts

After logging in to the admin panel, add proxy hosts for your internal services:

#### Example: Proxying Nextcloud

1. Click **Proxy Hosts** > **Add Proxy Host**
2. Configure:
   - **Domain Names:** `cloud.yourdomain.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `nextcloud.nextcloud.svc.cluster.local`
   - **Forward Port:** `80`
   - **Block Common Exploits:** Enabled
   - **Websockets Support:** Enabled
3. Under the **SSL** tab:
   - **SSL Certificate:** Request a new SSL Certificate
   - **Force SSL:** Enabled
   - **HTTP/2 Support:** Enabled

#### Common Internal Service URLs

| Service | Forward URL |
|---------|-------------|
| Nextcloud | `http://nextcloud.nextcloud.svc.cluster.local:80` |
| Home Assistant | `http://homeassistant.homeassistant.svc.cluster.local:8123` |
| Immich | `http://immich-server.immich.svc.cluster.local:2283` |
| Odoo | `http://odoo.odoo.svc.cluster.local:8069` |
| n8n | `http://n8n.n8n.svc.cluster.local:5678` |
| Open WebUI | `http://open-webui.open-webui.svc.cluster.local:8080` |
| Portainer | `https://portainer.portainer.svc.cluster.local:9443` |
| AnythingLLM | `http://anythingllm.anythingllm.svc.cluster.local:3001` |
| New API | `http://new-api.new-api.svc.cluster.local:3000` |
| EMQX Dashboard | `http://emqx.emqx.svc.cluster.local:18083` |

### Backup & Restore

#### Backup

```bash
# 1. Backup NPM data (configs, access lists, database)
kubectl -n nginxpm exec deploy/nginxpm -- tar czf /tmp/nginxpm-data-backup.tar.gz /data
kubectl -n nginxpm cp nginxpm/<pod-name>:/tmp/nginxpm-data-backup.tar.gz ./nginxpm-data-backup.tar.gz

# 2. Backup SSL certificates
kubectl -n nginxpm exec deploy/nginxpm -- tar czf /tmp/letsencrypt-backup.tar.gz /etc/letsencrypt
kubectl -n nginxpm cp nginxpm/<pod-name>:/tmp/letsencrypt-backup.tar.gz ./letsencrypt-backup.tar.gz
```

#### Restore

```bash
# 1. Restore NPM data
kubectl -n nginxpm cp ./nginxpm-data-backup.tar.gz nginxpm/<pod-name>:/tmp/nginxpm-data-backup.tar.gz
kubectl -n nginxpm exec deploy/nginxpm -- tar xzf /tmp/nginxpm-data-backup.tar.gz -C /

# 2. Restore SSL certificates
kubectl -n nginxpm cp ./letsencrypt-backup.tar.gz nginxpm/<pod-name>:/tmp/letsencrypt-backup.tar.gz
kubectl -n nginxpm exec deploy/nginxpm -- tar xzf /tmp/letsencrypt-backup.tar.gz -C /

# 3. Restart to pick up restored configuration
kubectl -n nginxpm rollout restart deploy/nginxpm
```

### Useful Commands

```bash
# Check all resources in the namespace
kubectl -n nginxpm get all

# View real-time logs
kubectl -n nginxpm logs deploy/nginxpm -f

# Restart Nginx Proxy Manager
kubectl -n nginxpm rollout restart deploy/nginxpm

# Check service endpoints
kubectl -n nginxpm get svc

# Test upstream connectivity from NPM pod
kubectl -n nginxpm exec deploy/nginxpm -- wget -qO- http://nextcloud.nextcloud.svc.cluster.local:80

# Delete and redeploy
kubectl delete -k k8s-manifests/nginxpm/
kubectl apply -k k8s-manifests/nginxpm/
```

### Troubleshooting

#### Cannot access admin panel

```bash
kubectl -n nginxpm get pods
kubectl -n nginxpm logs deploy/nginxpm

# Verify the service is exposed
kubectl -n nginxpm get svc
```

#### SSL certificate renewal fails

```bash
# Check Let's Encrypt logs inside the container
kubectl -n nginxpm exec deploy/nginxpm -- cat /data/logs/letsencrypt.log

# Common causes:
# - Port 80 not publicly accessible (required for HTTP-01 challenge)
# - DNS not pointing to the correct IP
# - Rate limiting by Let's Encrypt
```

#### 502 Bad Gateway errors

This typically means Nginx Proxy Manager cannot reach the upstream service:

```bash
# Test the upstream service from within the NPM pod
kubectl -n nginxpm exec deploy/nginxpm -- wget -qO- http://nextcloud.nextcloud.svc.cluster.local:80

# Verify the upstream service is running
kubectl get pods --all-namespaces | grep <service-name>
```

#### Default credentials not working

If you changed the password and forgot it, access the SQLite database:

```bash
kubectl -n nginxpm exec -it deploy/nginxpm -- sqlite3 /data/database.sqlite
# DELETE FROM auth WHERE id=1;
# Then restart and re-register with admin@example.com / changeme
```

#### Websocket connections failing

Ensure **Websockets Support** is enabled in the proxy host configuration. This is required for services like Home Assistant, n8n, and Immich.

### File Structure

```
k8s-manifests/nginxpm/
├── kustomization.yaml          # Kustomize entry point
├── namespace.yaml              # Namespace: nginxpm
├── nginxpm-deployment.yaml     # NPM Deployment
├── nginxpm-service.yaml        # NodePort service (30080, 30443, 30081)
├── pvc.yaml                    # PVCs for data and Let's Encrypt certs
└── README.md                   # This file
```

---

## 中文

### 概述

簡單易用的反向代理，具備自動 SSL 憑證管理（Let's Encrypt）和網頁圖形介面。Nginx Proxy Manager 簡化了將內部服務暴露至網際網路的流程，提供 SSL 終端、自訂網域、存取清單和重新導向規則。它透過簡潔的網頁介面管理代理主機，無需手動編輯 Nginx 設定檔。

> **GitHub 儲存庫 (Podman/Docker):** [Woow_nginxpm_docker_compose_all](https://github.com/WOOWTECH/Woow_nginxpm_docker_compose_all)

### 架構圖

```
                         ┌──────────────────────────────────────────────────────┐
                         │                  K3s / Kubernetes                    │
                         │                                                      │
                         │  ┌──────────────────────────────────────────────┐    │
  ┌───────────┐          │  │           命名空間: nginxpm                   │    │
  │   瀏覽器   │          │  │                                              │    │
  │  （管理）  │  :30081  │  │  ┌────────────────┐    ┌─────────────────┐  │    │
  │           ├──────────►│  │  │ Service        │    │  Deployment     │  │    │
  └───────────┘  NodePort │  │  │ nginxpm :81    ├───►│  nginxpm        │  │    │
                         │  │  └────────────────┘    │ (jc21/nginx-    │  │    │
  ┌───────────┐          │  │                         │  proxy-manager) │  │    │
  │  HTTP      │  :30080  │  │  ┌────────────────┐    │                 │  │    │
  │  流量      ├──────────►│  │  │ Service        │    │  :80  (HTTP)    │  │    │
  └───────────┘  NodePort │  │  │ nginxpm :80    ├───►│  :443 (HTTPS)   │  │    │
                         │  │  └────────────────┘    │  :81  (管理面板) │  │    │
  ┌───────────┐          │  │                         │                 │  │    │
  │  HTTPS     │  :30443  │  │  ┌────────────────┐    │ [PVC: 5Gi 資料] │  │    │
  │  流量      ├──────────►│  │  │ Service        │    │ [PVC: 1Gi 憑證] │  │    │
  └───────────┘  NodePort │  │  │ nginxpm :443   ├───►│                 │  │    │
                         │  │  └────────────────┘    └────────┬────────┘  │    │
                         │  │                                  │           │    │
                         │  └──────────────────────────────────┼───────────┘    │
                         │                                      │               │
                         │           代理的上游服務               ▼               │
                         │  ┌───────────────────────────────────────────────┐   │
                         │  │  nextcloud.nextcloud.svc:80                   │   │
                         │  │  odoo.odoo.svc:8069                           │   │
                         │  │  n8n.n8n.svc:5678                             │   │
                         │  │  immich-server.immich.svc:2283                │   │
                         │  │  homeassistant.homeassistant.svc:8123         │   │
                         │  │  ...（任何叢集服務）                            │   │
                         │  └───────────────────────────────────────────────┘   │
                         └──────────────────────────────────────────────────────┘

  連接埠對應:
    外部 :30080  ──►  Service :80   ──►  Pod nginxpm :80   (HTTP 代理)
    外部 :30443  ──►  Service :443  ──►  Pod nginxpm :443  (HTTPS 代理)
    外部 :30081  ──►  Service :81   ──►  Pod nginxpm :81   (管理面板)
```

### 功能特色

- 網頁圖形介面管理反向代理設定
- 透過 Let's Encrypt 自動佈建 SSL 憑證
- SSL 終端，支援 HTTP/2
- 存取清單提供 IP 限制功能
- 支援自訂 Nginx 設定
- 重新導向與串流規則
- Websocket 代理支援（Home Assistant、n8n 等服務所需）
- 內建 SQLite 資料庫（無需外部資料庫）

### 快速開始

```bash
# 1. 部署 Nginx Proxy Manager
kubectl apply -k k8s-manifests/nginxpm/

# 2. 確認 Pod 正常運行
kubectl -n nginxpm get pods

# 3. 存取管理面板
echo "http://<節點IP>:30081"
```

### 設定

#### 無需環境變數或密鑰

Nginx Proxy Manager 完全透過網頁介面進行設定。此部署不使用 ConfigMap 或 Secret 檔案。

#### 預設管理員帳號

| 設定 | 值 |
|------|---|
| 電子郵件 | `admin@example.com` |
| 密碼 | `changeme` |

**首次登入後請立即更改。**

### 存取服務

| 端點 | URL | 協定 |
|------|-----|------|
| 管理面板 | `http://<節點IP>:30081` | HTTP (NodePort) |
| HTTP 代理 | `http://<節點IP>:30080` | HTTP (NodePort) |
| HTTPS 代理 | `https://<節點IP>:30443` | HTTPS (NodePort) |
| 內部管理 | `http://nginxpm.nginxpm.svc.cluster.local:81` | HTTP |
| 內部 HTTP | `http://nginxpm.nginxpm.svc.cluster.local:80` | HTTP |
| 內部 HTTPS | `https://nginxpm.nginxpm.svc.cluster.local:443` | HTTPS |

#### 連接埠摘要

| 連接埠 | NodePort | 用途 |
|--------|----------|------|
| 80 | 30080 | HTTP 流量（代理） |
| 443 | 30443 | HTTPS 流量（含 SSL 的代理） |
| 81 | 30081 | 管理面板網頁介面 |

### 資料持久化

| PVC 名稱 | 掛載路徑 | 大小 | 用途 |
|----------|----------|------|------|
| `nginxpm-data` | `/data` | 5Gi | 代理設定、存取清單、自訂位置、SQLite 資料庫 |
| `nginxpm-letsencrypt` | `/etc/letsencrypt` | 1Gi | Let's Encrypt SSL 憑證與 ACME 帳號資料 |

所有 PVC 使用 `local-path` 儲存類別（k3s 預設）。

### 設定代理主機

登入管理面板後，為內部服務新增代理主機：

#### 範例：代理 Nextcloud

1. 點擊 **Proxy Hosts** > **Add Proxy Host**
2. 設定：
   - **Domain Names:** `cloud.yourdomain.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `nextcloud.nextcloud.svc.cluster.local`
   - **Forward Port:** `80`
   - **Block Common Exploits:** 啟用
   - **Websockets Support:** 啟用
3. 在 **SSL** 標籤下：
   - **SSL Certificate:** Request a new SSL Certificate
   - **Force SSL:** 啟用
   - **HTTP/2 Support:** 啟用

#### 常用內部服務 URL

| 服務 | 轉發 URL |
|------|----------|
| Nextcloud | `http://nextcloud.nextcloud.svc.cluster.local:80` |
| Home Assistant | `http://homeassistant.homeassistant.svc.cluster.local:8123` |
| Immich | `http://immich-server.immich.svc.cluster.local:2283` |
| Odoo | `http://odoo.odoo.svc.cluster.local:8069` |
| n8n | `http://n8n.n8n.svc.cluster.local:5678` |
| Open WebUI | `http://open-webui.open-webui.svc.cluster.local:8080` |
| Portainer | `https://portainer.portainer.svc.cluster.local:9443` |
| AnythingLLM | `http://anythingllm.anythingllm.svc.cluster.local:3001` |
| New API | `http://new-api.new-api.svc.cluster.local:3000` |
| EMQX Dashboard | `http://emqx.emqx.svc.cluster.local:18083` |

### 備份與還原

#### 備份

```bash
# 1. 備份 NPM 資料（設定、存取清單、資料庫）
kubectl -n nginxpm exec deploy/nginxpm -- tar czf /tmp/nginxpm-data-backup.tar.gz /data
kubectl -n nginxpm cp nginxpm/<pod-name>:/tmp/nginxpm-data-backup.tar.gz ./nginxpm-data-backup.tar.gz

# 2. 備份 SSL 憑證
kubectl -n nginxpm exec deploy/nginxpm -- tar czf /tmp/letsencrypt-backup.tar.gz /etc/letsencrypt
kubectl -n nginxpm cp nginxpm/<pod-name>:/tmp/letsencrypt-backup.tar.gz ./letsencrypt-backup.tar.gz
```

#### 還原

```bash
# 1. 還原 NPM 資料
kubectl -n nginxpm cp ./nginxpm-data-backup.tar.gz nginxpm/<pod-name>:/tmp/nginxpm-data-backup.tar.gz
kubectl -n nginxpm exec deploy/nginxpm -- tar xzf /tmp/nginxpm-data-backup.tar.gz -C /

# 2. 還原 SSL 憑證
kubectl -n nginxpm cp ./letsencrypt-backup.tar.gz nginxpm/<pod-name>:/tmp/letsencrypt-backup.tar.gz
kubectl -n nginxpm exec deploy/nginxpm -- tar xzf /tmp/letsencrypt-backup.tar.gz -C /

# 3. 重啟以載入還原的設定
kubectl -n nginxpm rollout restart deploy/nginxpm
```

### 實用指令

```bash
# 檢視命名空間中的所有資源
kubectl -n nginxpm get all

# 即時檢視日誌
kubectl -n nginxpm logs deploy/nginxpm -f

# 重啟 Nginx Proxy Manager
kubectl -n nginxpm rollout restart deploy/nginxpm

# 檢查服務端點
kubectl -n nginxpm get svc

# 從 NPM Pod 測試上游連線
kubectl -n nginxpm exec deploy/nginxpm -- wget -qO- http://nextcloud.nextcloud.svc.cluster.local:80

# 刪除並重新部署
kubectl delete -k k8s-manifests/nginxpm/
kubectl apply -k k8s-manifests/nginxpm/
```

### 疑難排解

#### 無法存取管理面板

```bash
kubectl -n nginxpm get pods
kubectl -n nginxpm logs deploy/nginxpm

# 確認服務已暴露
kubectl -n nginxpm get svc
```

#### SSL 憑證更新失敗

```bash
# 檢查容器內的 Let's Encrypt 日誌
kubectl -n nginxpm exec deploy/nginxpm -- cat /data/logs/letsencrypt.log

# 常見原因：
# - 連接埠 80 未公開（HTTP-01 驗證所需）
# - DNS 未指向正確的 IP
# - Let's Encrypt 速率限制
```

#### 502 Bad Gateway 錯誤

這通常表示 Nginx Proxy Manager 無法連線到上游服務：

```bash
# 從 NPM Pod 內部測試上游服務
kubectl -n nginxpm exec deploy/nginxpm -- wget -qO- http://nextcloud.nextcloud.svc.cluster.local:80

# 確認上游服務正在運行
kubectl get pods --all-namespaces | grep <service-name>
```

#### 預設帳號密碼不可用

如果已更改密碼且遺忘，可存取 SQLite 資料庫：

```bash
kubectl -n nginxpm exec -it deploy/nginxpm -- sqlite3 /data/database.sqlite
# DELETE FROM auth WHERE id=1;
# 然後重啟並使用 admin@example.com / changeme 重新註冊
```

#### Websocket 連線失敗

請確保在代理主機設定中啟用 **Websockets Support**。Home Assistant、n8n 和 Immich 等服務需要此功能。

### 檔案結構

```
k8s-manifests/nginxpm/
├── kustomization.yaml          # Kustomize 進入點
├── namespace.yaml              # 命名空間: nginxpm
├── nginxpm-deployment.yaml     # NPM Deployment
├── nginxpm-service.yaml        # NodePort 服務（30080、30443、30081）
├── pvc.yaml                    # 資料與 Let's Encrypt 憑證的 PVC
└── README.md                   # 本文件
```
