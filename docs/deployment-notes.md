# 📋 Production Deployment Notes

> Everything we did to take this project from code to **live production** on AWS EC2.  
> Follow these steps in exact order if you ever need to deploy this project again from scratch.  
> **Note:** `notes.md` covers the project setup (TypeScript, ESLint, Prettier, Husky, basic Docker concepts).  
> This file covers the **production-specific Docker configs, CI/CD, EC2 setup, SSL, monitoring, and deployment steps**.

---

## Architecture

```
                    Internet
                       │
                       ▼
              ┌────────────────┐
              │  AWS EC2       │
              │  t3.medium     │
              │  4GB RAM       │
              │  Ubuntu 24.04  │
              │                │
              │ ┌────────────┐ │
         :80 ─┤►│   Nginx    │ │
        :443 ─┤►│  (reverse  │ │
              │ │   proxy +  │ │
              │ │ rate limit │ │
              │ │  + SSL)    │ │
              │ └─────┬──────┘ │
              │       │        │
              │       ▼        │
              │ ┌────────────┐ │
              │ │   App      │ │
              │ │  (Node.js) │ │
              │ │  Express   │ │
              │ │  Winston   │ │
              │ └─────┬──────┘ │
              │       │        │
              │       ▼        │
              │ ┌────────────┐ │
              │ │   Redis    │ │
              │ │  (cache +  │ │
              │ │  rate limit)│ │
              │ └────────────┘ │
              │                │
              │ ┌────────────┐ │      ┌─────────────────────┐
              │ │  Alloy     │─┼─────►│ Grafana Cloud Loki  │
              │ │  (log      │ │      │ (log search +       │
              │ │  shipper)  │ │      │  alerts)            │
              │ └────────────┘ │      └─────────────────────┘
              │                │
              │ ┌────────────┐ │
              │ │  Certbot   │ │
              │ │  (SSL)     │ │
              │ └────────────┘ │
              │                │
              └────────────────┘
                       │
              ┌────────┴────────┐     ┌─────────────────┐
              │ AWS CloudWatch  │     │ UptimeRobot     │
              │ (server metrics)│     │ (uptime check)  │
              └─────────────────┘     └─────────────────┘
```

### What Each Component Does

| Component   | Purpose                                                         | RAM          | Docker Image           |
| ----------- | --------------------------------------------------------------- | ------------ | ---------------------- |
| **Nginx**   | SSL termination, reverse proxy, rate limiting, security headers | ~10MB        | `nginx:stable-alpine`  |
| **App**     | Node.js API (Express + Winston structured JSON logging)         | ~150MB       | Custom `Dockerfile`    |
| **Redis**   | In-memory cache, rate limit counters                            | ~50-256MB    | `redis:7-alpine`       |
| **Alloy**   | Reads Docker logs → parses JSON → ships to Grafana Cloud Loki   | ~50MB        | `grafana/alloy:v1.8.3` |
| **Certbot** | Obtains and renews Let's Encrypt SSL certificates               | Runs briefly | `certbot/certbot`      |

### How Code Gets to Production

```
Feature Branch → PR → CI Checks → Merge → Build Image → Deploy

┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  ci.yaml    │     │  build.yaml  │     │  deploy.yaml │
│  (auto)     │     │  (manual)    │     │  (manual)    │
│             │     │              │     │              │
│  Triggers:  │     │  Triggers:   │     │  Triggers:   │
│  PR to main │     │  Manual      │     │  Manual      │
│             │     │              │     │              │
│  ✓ Lint     │     │  ✓ Build     │     │  ✓ SSH EC2   │
│    (3 Node  │     │    Docker    │     │  ✓ git pull  │
│    versions)│     │    image     │     │  ✓ docker    │
│  ✓ Build    │     │  ✓ Push to   │     │    compose   │
│    (3 Node  │     │    Docker    │     │    pull      │
│    versions)│     │    Hub       │     │  ✓ docker    │
│  ✓ Tests    │     │  ✓ Update    │     │    compose   │
│    (with    │     │    compose   │     │    up -d     │
│    Redis)   │     │    .yaml     │     │  ✓ Health    │
│             │     │  ✓ Commit    │     │    check     │
│             │     │    back      │     │  ✓ Cleanup   │
└─────────────┘     └──────────────┘     └──────────────┘
```

---

## Step 1: Nginx Configuration

### 1.1 Main Config (`nginx/nginx.conf`)

```nginx
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 10M;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml;

    # Rate Limiting Zones
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;     # 10 req/s per IP
    limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m;     # 5 req/min per IP (brute force)

    include /etc/nginx/conf.d/*.conf;
}
```

### 1.2 Server Config (`nginx/conf.d/default.conf`)

**Before SSL (initial deployment):**

```nginx
upstream api_backend {
    server app:3000;
}

server {
    listen 80;
    server_name anuragsahu.duckdns.org;

    # Let's Encrypt ACME challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # API routes — rate limited
    location / {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Health check — no rate limiting
    location /health {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**After getting SSL certificate (replace entire file with this):**

```nginx
upstream api_backend {
    server app:3000;
}

# HTTP → Redirect to HTTPS
server {
    listen 80;
    server_name anuragsahu.duckdns.org;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS Server (SSL + Reverse Proxy + Rate Limiting)
server {
    listen 443 ssl;
    server_name anuragsahu.duckdns.org;

    # SSL Certificates
    ssl_certificate /etc/letsencrypt/live/anuragsahu.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/anuragsahu.duckdns.org/privkey.pem;

    # SSL Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # API routes — rate limited
    location / {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Health check — no rate limiting
    location /health {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## Step 2: Grafana Cloud + Alloy (Log Shipping)

### 2.1 Create Grafana Cloud Account

1. Go to [grafana.com/cloud](https://grafana.com/products/cloud/) → Sign up (free tier: 50GB/month)
2. Go to **My Account → Loki section**
3. Copy the **URL**: `https://logs-prod-xxx.grafana.net/loki/api/v1/push`
4. Copy the **Username**: (numeric ID like `1493096`)
5. Go to **Security → API Keys → Add API Key**
   - Name: `alloy`
   - Role: `MetricsPublisher`
   - **Copy the key** (shown only once!)

These 3 values go into your `.env` on the server:

```env
LOKI_URL=https://logs-prod-028.grafana.net/loki/api/v1/push
LOKI_USERNAME=1493096
LOKI_API_KEY=glc_xxxxxxxxxxxx
```

### 2.2 Alloy Configuration (`alloy/config.alloy`)

```hcl
// ─── Discover Docker container log files ───
local.file_match "docker_logs" {
  path_targets = [{"__path__" = "/var/lib/docker/containers/*/*-json.log"}]
}

loki.source.file "docker" {
  targets    = local.file_match.docker_logs.targets
  forward_to = [loki.process.docker.receiver]
}

// ─── Parse Docker + Winston JSON logs ───
loki.process "docker" {
  // Unwrap Docker's JSON wrapper (extracts the actual log line)
  stage.docker {}

  // Parse Winston JSON fields (level, service)
  stage.json {
    expressions = {
      level   = "level",
      service = "service",
    }
  }

  // Promote to Loki labels (so you can filter by level="error", service="api")
  stage.labels {
    values = {
      level   = "",
      service = "",
    }
  }

  // Static labels for identification
  stage.static_labels {
    values = {
      job  = "docker",
      host = "my-api-server",
    }
  }

  forward_to = [loki.write.grafana_cloud.receiver]
}

// ─── Ship to Grafana Cloud Loki ───
loki.write "grafana_cloud" {
  endpoint {
    url = env("LOKI_URL")

    basic_auth {
      username = env("LOKI_USERNAME")
      password = env("LOKI_API_KEY")
    }
  }
}
```

**How the log pipeline works:**

```
App (Winston JSON) → Docker stdout → Docker log file → Alloy reads file → Alloy parses JSON → Alloy extracts level + service → Ships to Grafana Cloud Loki
```

### 2.3 Useful Loki Queries in Grafana

Go to **Grafana Cloud → Explore (compass icon 🧭) → Select Loki datasource → Switch to "Code" mode**

| What you want     | Query                                |
| ----------------- | ------------------------------------ |
| All logs          | `{host="my-api-server"}`             |
| Only API app logs | `{service="api"}`                    |
| Only errors       | `{level="error"}`                    |
| Only warnings     | `{level="warn"}`                     |
| Search for text   | `{host="my-api-server"} \|= "redis"` |
| Nginx logs        | `{host="my-api-server"} \|= "nginx"` |
| HTTP 500 errors   | `{host="my-api-server"} \|= "500"`   |

---

## Step 3: Production Docker Configs

> `notes.md` has the initial/basic Dockerfiles. Below are the **final production versions** we actually use — they're different.

### 3.1 Production Dockerfile (`Dockerfile`)

Multi-stage build — builder is heavy (~200MB), runner is lightweight (~50MB):

```dockerfile
# Stage 1: Build
FROM node:22-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --ignore-scripts
COPY . .
RUN npm run build
RUN npm prune --omit=dev

# Stage 2: Run
FROM node:22-slim AS runner
WORKDIR /app
RUN useradd -m appuser
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/dist ./dist
USER appuser
EXPOSE 3000
CMD ["node", "--enable-source-maps", "dist/index.js"]
```

**Differences from the basic version in `notes.md`:**

- `node:22-slim` instead of `node:22-alpine` (slim has glibc — fewer compatibility issues)
- `npm ci --ignore-scripts` (skips post-install scripts for security)
- `npm prune --omit=dev` **in builder** (removes devDependencies before COPY to runner — smaller image)
- `useradd -m appuser` instead of `addgroup/adduser` (slim is Debian-based, not Alpine)
- `--enable-source-maps` (error traces show `.ts` line numbers instead of compiled `.js`)
- No `HEALTHCHECK` in Dockerfile — we define it in `compose.yaml` instead (more flexible)

### 3.2 `.dockerignore`

Keeps Docker build context small and fast:

```
node_modules
dist
npm-debug.log
.env
.git
.gitignore
*.md
tests
.prettierrc
.prettierignore
eslint.config.js
```

### 3.3 `.env.example`

Template for the `.env` file (committed to Git):

```env
NODE_ENV=development
PORT=3000
CORS_ORIGIN=http://localhost:5173
REDIS_URL=redis://redis:6379

# Grafana Cloud — Loki (logs)
LOKI_URL=https://logs-prod-xxx.grafana.net/loki/api/v1/push
LOKI_USERNAME=your-loki-user-id
LOKI_API_KEY=your-grafana-api-key
```

### 3.4 `.gitignore`

```
node_modules/
dist/
.env
*.log
```

---

## Step 4: GitHub Setup

### 4.1 Create GitHub Repository

```bash
git init
git add .
git commit -m "chore: initial commit"
git branch -M main
git remote add origin https://github.com/anuragsahu-dev/Production-Deploy.git
git push -u origin main
```

### 4.2 Create GitHub App (needed for build.yaml to push compose.yaml updates)

1. Go to **GitHub → Settings → Developer Settings → GitHub Apps → New GitHub App**
2. Fill in:
   - **Name:** `deploy-bot` (must be globally unique)
   - **Homepage URL:** your repo URL
   - **Permissions → Repository → Contents:** `Read & Write`
   - **Where can this app be installed?** `Only on this account`
3. Click **Create GitHub App**
4. Note the **App ID** (number at top of the app page)
5. Scroll down → **Generate a private key** → downloads a `.pem` file
6. Go to **Install App** (left sidebar) → Install on your repository

### 4.3 Branch Protection (Rulesets)

Go to **Repo → Settings → Rules → Rulesets → New ruleset**:

| Rule                      | Setting         |
| ------------------------- | --------------- |
| Require pull request      | ✅ Yes          |
| Require 0 approvals       | (for solo dev)  |
| Require status checks     | ✅ CI must pass |
| Require up-to-date branch | ✅ Yes          |
| Block force pushes        | ✅ Yes          |
| Restrict deletions        | ✅ Yes          |

**Important:** Add your **GitHub App (`deploy-bot`)** to the **Bypass list** with "Always" permission. This allows the build workflow bot to push `compose.yaml` tag updates directly to `main` without a PR.

### 4.4 Docker Hub Setup

1. Create account at [hub.docker.com](https://hub.docker.com)
2. Go to **Account Settings → Security → New Access Token**
   - Description: `github-actions`
   - Permissions: `Read & Write`
   - Copy the token (shown once)
3. Create a repository: **Create Repository** → Name: `api` → Visibility: Private

### 4.5 Add All GitHub Secrets

Go to **Repo → Settings → Secrets and variables → Actions → New repository secret**:

| Secret               | Value                                        | Used By     |
| -------------------- | -------------------------------------------- | ----------- |
| `DOCKERHUB_USERNAME` | Your Docker Hub username (e.g., `as3305100`) | build.yaml  |
| `DOCKERHUB_PASSWORD` | Docker Hub access token (NOT your password)  | build.yaml  |
| `APP_ID`             | GitHub App ID (number)                       | build.yaml  |
| `APP_SECRET_KEY`     | Full contents of the `.pem` file             | build.yaml  |
| `EC2_HOST`           | Your Elastic IP address                      | deploy.yaml |
| `EC2_USERNAME`       | `ubuntu`                                     | deploy.yaml |
| `EC2_SSH_KEY`        | Full contents of `my-api-key.pem`            | deploy.yaml |
| `EC2_SSH_PORT`       | `22` (or your custom port)                   | deploy.yaml |

### 4.6 Update IMAGE_NAME in 3 Files

Replace `YOUR_DOCKERHUB_USERNAME/api` with your actual Docker Hub image name (e.g., `as3305100/api`) in:

| File                            | Where                 |
| ------------------------------- | --------------------- |
| `compose.yaml`                  | Line 3: `image:`      |
| `.github/workflows/build.yaml`  | Line 7: `IMAGE_NAME:` |
| `.github/workflows/deploy.yaml` | Line 7: `IMAGE_NAME:` |

---

## Step 5: GitHub Actions Workflows

### 5.1 `ci.yaml` — PR Verification (Automatic)

Triggers on every Pull Request to `main`. Runs:

- **Lint** across Node 20, 22, 24
- **Build** across Node 20, 22, 24
- **Tests** with a Redis service container (Node 22)

### 5.2 `build.yaml` — Build & Push Docker Image (Manual Trigger)

What it does step by step:

1. Checks out the code
2. Generates image tag from commit SHA (e.g., `build-8b4bc7d...`)
3. Updates `compose.yaml` with new tag using `yaml-update-action`
4. Generates a GitHub App token (to bypass branch protection)
5. Commits and pushes the updated `compose.yaml` back to `main`
6. Logs into Docker Hub
7. Builds Docker image with `--platform linux/amd64`
8. Tags as both `build-<SHA>` and `latest`
9. Pushes both tags to Docker Hub

### 5.3 `deploy.yaml` — Deploy to EC2 (Manual Trigger)

What it does step by step:

1. Checks out code
2. Extracts image tag from `compose.yaml`
3. SSHs into EC2 using `appleboy/ssh-action`
4. **First deployment**: Backs up `.env` → Clones repo → Restores `.env`
5. **Subsequent deploys**: `git fetch` + `git reset --hard origin/main`
6. Pulls the Docker image from Docker Hub
7. Runs `docker compose up -d`
8. **Health check loop**: Checks every 5s for up to 120s, waiting for `healthy` status
9. If timeout: prints container status + last 50 lines of app logs + exits with error
10. If healthy: prints success + all container statuses
11. Cleans up old Docker images

**Bug we fixed:** The health check originally used `docker inspect api` but since we removed `container_name`, we changed it to `docker compose ps app --format '{{.Health}}'`.

**Bug we fixed:** `.env` was getting deleted on first deploy because `rm -rf ~/app` was called before `git clone`. We added backup/restore logic.

---

## Step 6: AWS EC2 Setup

### 6.1 Launch EC2 Instance

| Setting       | Value                                       |
| ------------- | ------------------------------------------- |
| Name          | `my-api-server`                             |
| AMI           | Ubuntu Server 24.04 LTS                     |
| Instance Type | `t3.small` (2GB) or `t3.medium` (4GB)       |
| Key Pair      | Create new → `my-api-key` → Download `.pem` |
| Storage       | 20 GB gp3 SSD                               |

### 6.2 Security Group (Firewall)

| Type  | Port | Source                    | Purpose        |
| ----- | ---- | ------------------------- | -------------- |
| SSH   | 22   | 0.0.0.0/0 (with key auth) | Admin access   |
| HTTP  | 80   | 0.0.0.0/0                 | Web + Certbot  |
| HTTPS | 443  | 0.0.0.0/0                 | Secure traffic |

> ❌ Do NOT open ports 3000 or 6379. App and Redis are internal only.

**Note on SSH security:** We chose Option 3 (SSH open to 0.0.0.0/0 with key-only authentication) for simplicity. In future, consider Option 1 (dynamic security group updates via IAM) for tighter security.

### 6.3 Allocate Elastic IP

1. **EC2 → Elastic IPs → Allocate**
2. **Actions → Associate** → select your instance
3. Update `EC2_HOST` GitHub secret with this IP

Without Elastic IP, the public IP changes every time EC2 starts/stops.

### 6.4 SSH into EC2

```bash
# Fix .pem permissions (Windows PowerShell — one time)
icacls my-api-key.pem /reset
icacls my-api-key.pem /grant:r "%username%:R"
icacls my-api-key.pem /inheritance:r

# Connect
ssh -i my-api-key.pem ubuntu@<ELASTIC_IP>
```

---

## Step 7: Install Docker on EC2 (Ubuntu 24.04 LTS)

We used Docker's **officially recommended** apt repository method (not `get.docker.com`).  
This gives proper package management — future updates come automatically via `sudo apt upgrade`.

Run these commands **on the EC2 server** (after SSH):

### 7.1 Update system

```bash
sudo apt update && sudo apt upgrade -y
```

### 7.2 Add Docker's official GPG key

```bash
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### 7.3 Add Docker repository to apt sources

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```

### 7.4 Install Docker Engine + Compose plugin

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### 7.5 Add user to docker group (so you don't need sudo)

```bash
sudo usermod -aG docker $USER

# Log out and back in for group changes to take effect
exit
```

### 7.6 Reconnect and verify

```bash
ssh -i my-api-key.pem ubuntu@<ELASTIC_IP>

docker --version          # Should show Docker version (e.g., 27.x)
docker compose version    # Should show Compose version (e.g., v2.x)
sudo docker run hello-world   # Should print "Hello from Docker!"
```

### 7.7 Configure Docker Log Rotation

Without this, logs grow forever and fill the disk.

```bash
sudo nano /etc/docker/daemon.json
```

Paste:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Save (`Ctrl+O → Enter → Ctrl+X`) and restart Docker:

```bash
sudo systemctl restart docker
```

Limits each container to 30MB of logs (3 files × 10MB).

### 7.8 Create `.env` File on EC2

```bash
mkdir -p ~/app
nano ~/app/.env
```

Paste your production secrets:

```env
PORT=3000
CORS_ORIGIN=https://anuragsahu.duckdns.org
REDIS_URL=redis://redis:6379

LOKI_URL=https://logs-prod-028.grafana.net/loki/api/v1/push
LOKI_USERNAME=1493096
LOKI_API_KEY=glc_xxxxxxxxxxxx
```

> This file is NEVER in Git. It only lives on the server. The deploy workflow automatically backs it up and restores it during deploys.

Save: `Ctrl+O → Enter → Ctrl+X`

---

## Step 8: Domain Setup (DuckDNS)

### 8.1 Point Domain to EC2

1. Go to [duckdns.org](https://www.duckdns.org/) → Log in
2. Set `anuragsahu` subdomain → your **Elastic IP**
3. Click **Update**

### 8.2 Verify

```bash
ping anuragsahu.duckdns.org   # Should resolve to your Elastic IP
curl http://anuragsahu.duckdns.org/health   # Should return health JSON
```

---

## Step 9: First Deployment

### 9.1 Via GitHub Actions (Recommended)

1. Go to **GitHub → Actions → "Build Docker Image" → Run workflow**
2. Wait for it to complete ✅ (builds image, pushes to Docker Hub, updates compose.yaml)
3. **Pull the latest `main`** locally (the bot just pushed a compose.yaml update)
4. Go to **GitHub → Actions → "Deploy" → Run workflow**
5. Wait for the health check to pass ✅

### 9.2 What Happened During Our First Deploy

- SSH connection timed out initially → Fixed by opening SSH in security group
- `.env` got deleted during `rm -rf ~/app` + `git clone` → Fixed by adding backup/restore logic
- Image tag grep was matching Redis/Nginx lines → Fixed by specifically matching `as3305100/api:`
- Health check used `docker inspect api` → Fixed to use `docker compose ps app --format '{{.Health}}'`

### 9.3 Verify on EC2

```bash
# From your browser
http://<ELASTIC_IP>/health

# From EC2
docker compose ps   # All 5 containers should be running
```

---

## Step 10: SSL Certificate (Certbot)

### 10.1 Create Certbot Directories

```bash
cd ~/app
mkdir -p certbot/conf certbot/www
```

### 10.2 Get the Certificate

```bash
docker compose run --rm certbot certonly \
  --webroot --webroot-path=/var/www/certbot \
  -d anuragsahu.duckdns.org \
  --email your-email@gmail.com --agree-tos --no-eff-email
```

Should see: `Successfully received certificate.`

### 10.3 Enable HTTPS in Nginx

Edit `~/app/nginx/conf.d/default.conf`:

- Uncomment the HTTPS server block (port 443)
- Replace all `your-domain.com` with `anuragsahu.duckdns.org`
- In the HTTP block (port 80), change `location /` to redirect:
  ```nginx
  location / {
      return 301 https://$host$request_uri;
  }
  ```

### 10.4 Restart Nginx

```bash
docker compose restart nginx
# or
docker compose up -d nginx
```

### 10.5 Verify HTTPS

```bash
curl https://anuragsahu.duckdns.org/health
```

### 10.6 Auto-Renewal Cron

```bash
crontab -e
```

Add at the bottom:

```
0 3 * * * cd ~/app && docker compose run --rm certbot renew && docker compose exec nginx nginx -s reload
```

Runs daily at 3 AM. Only renews if cert is within 30 days of expiry. Certs expire every 90 days.

---

## Step 11: Monitoring Setup

### 11.1 UptimeRobot (Free — Uptime Alerts)

1. Go to [uptimerobot.com](https://uptimerobot.com) → Sign up
2. Click **+ Add New Monitor**:
   - **Monitor Type:** HTTPS
   - **Friendly Name:** `My API`
   - **URL:** `https://anuragsahu.duckdns.org/health`
   - **Monitoring Interval:** 5 minutes
3. Add your email as alert contact
4. Click **Create Monitor**

If 3 consecutive checks fail → you get an email notification.

### 11.2 Grafana Cloud Loki (Already set up in Step 2)

Alloy is running and shipping logs. Verify in Grafana Cloud:

1. Go to `your-org.grafana.net`
2. Click **Explore** (compass icon 🧭)
3. Select Loki data source
4. Query: `{host="my-api-server"}`
5. You should see your container logs

### 11.3 AWS CloudWatch (Automatic — no setup needed)

CloudWatch is enabled by default on EC2 — no setup needed.
View at: **AWS Console → CloudWatch → Metrics → EC2 → Per-Instance Metrics**

Optional alarms: Create alarm for **CPU > 80% for 5 min** and **Status Check Failed**.

### 11.4 Monitoring Summary

```
UptimeRobot     → "Is the app alive?"           → Pings /health every 5 min
Grafana Loki    → "What went wrong?"             → Search logs, filter by level/service
AWS CloudWatch  → "Is the server healthy?"       → CPU, RAM, disk, network

All three are FREE.
```

---

## Step 12: Everyday Workflow (After Setup is Complete)

```
1. Write code on a feature branch
2. git commit -m "feat: add something"      ← Husky: lint + format + commitlint
3. git push origin feature-branch           ← Husky: test + build
4. Create PR to main                        ← CI runs: lint, build, test (3 Node versions)
5. Merge PR (after CI passes)
6. GitHub Actions → "Build Docker Image"    ← Manual trigger
7. Pull latest main locally                 ← bot just pushed compose.yaml update
8. GitHub Actions → "Deploy"                ← Manual trigger
9. App is live! 🚀
```

---

## Step 13: Useful EC2 Commands

```bash
# Check container status
docker compose ps

# View real-time app logs
docker compose logs -f app

# View last 50 lines of a service
docker compose logs --tail 50 nginx

# View Alloy logs (check if log shipping works)
docker compose logs --tail 20 alloy

# Restart a specific service
docker compose restart app

# Restart everything
docker compose up -d

# Check disk usage
df -h

# Check memory usage
free -m

# Clean up unused Docker stuff
docker system prune -f
docker image prune -f
```

---

## Step 14: Scaling with Multiple Replicas (Optional)

### 14.1 Running Multiple API Containers

**Method 1 — Manual (temporary):**

```bash
docker compose up -d --scale app=3
```

**Method 2 — Permanent (in `compose.yaml`):**

```yaml
app:
  image: as3305100/api:latest
  deploy:
    replicas: 2 # Always run 2 instances
  restart: unless-stopped
  # ... rest of config
```

This creates multiple containers:

```
                    ┌── app-app-1 (:3000)
                    │
Nginx (:80/443) ────┼── app-app-2 (:3000)
                    │
                    └── app-app-3 (:3000)
```

### 14.2 Who Handles Load Balancing? → Nginx (Already Configured)

**Nginx is already your load balancer.** Here's why it works automatically:

Your `default.conf` has:

```nginx
upstream api_backend {
    server app:3000;
}
```

When you scale `app` to multiple replicas, Docker's **internal DNS** resolves `app` to **all container IPs**. Nginx then automatically **round-robins** requests across all of them.

**For better load balancing**, update the upstream block:

```nginx
upstream api_backend {
    least_conn;          # Send requests to the container with fewest active connections
    server app:3000;     # Docker DNS resolves to ALL app containers automatically
}
```

### 14.3 Load Balancing Strategies

| Strategy              | Nginx Directive | How it works                                                       |
| --------------------- | --------------- | ------------------------------------------------------------------ |
| **Round Robin**       | (default)       | Sends to each container in order: 1, 2, 3, 1, 2, 3...              |
| **Least Connections** | `least_conn;`   | Sends to the container with fewest active connections              |
| **IP Hash**           | `ip_hash;`      | Same client IP always goes to the same container (sticky sessions) |

### 14.4 RAM Considerations

| Replicas | API RAM | Total Server RAM | t3.small (2GB)? | t3.medium (4GB)? |
| -------- | ------- | ---------------- | --------------- | ---------------- |
| 1        | ~150MB  | ~450MB           | ✅ Comfortable  | ✅ Comfortable   |
| 2        | ~300MB  | ~600MB           | ✅ OK           | ✅ Comfortable   |
| 3        | ~450MB  | ~750MB           | ⚠️ Tight        | ✅ OK            |
| 4+       | ~600MB+ | ~900MB+          | ❌ Too much     | ⚠️ Tight         |

> Remember: Redis (~50-256MB), Nginx (~10MB), and Alloy (~50MB) also use RAM.  
> **Recommendation**: t3.small → max 1-2 replicas. Upgrade to t3.medium for 3+.

---

## Step 15: Project Files Reference

```
├── .github/workflows/
│   ├── ci.yaml              # PR checks: lint + build + test (3 Node versions)
│   ├── build.yaml           # Build Docker image → push to Docker Hub → update compose.yaml
│   └── deploy.yaml          # SSH to EC2 → pull → docker compose up -d → health check
│
├── .husky/
│   ├── pre-commit           # npm run lint + npm run format:check
│   ├── commit-msg           # commitlint (conventional commits)
│   └── pre-push             # npm run test + npm run build
│
├── alloy/
│   └── config.alloy         # Grafana Alloy: Docker logs → parse JSON → ship to Loki
│
├── nginx/
│   ├── nginx.conf           # Main config: gzip, rate limit zones (api + auth)
│   └── conf.d/default.conf  # Server block: upstream, proxy, SSL, security headers
│
├── src/
│   ├── config/
│   │   ├── env.ts           # Centralized env vars
│   │   ├── logger.ts        # Winston: JSON prod, colorized dev, service="api" label
│   │   └── redis.ts         # ioredis with reconnect strategy
│   ├── index.ts             # Express app + /health endpoint + graceful shutdown
│   ├── routes/              # API routes
│   ├── services/            # Business logic
│   └── data/                # Data layer
│
├── compose.yaml             # Production: 5 services (app, redis, nginx, certbot, alloy)
├── compose.dev.yaml         # Development: 2 services (app with hot reload, redis)
├── Dockerfile               # Multi-stage production build (builder → runner, non-root)
├── Dockerfile.dev           # Development with hot reload (tsx watch)
├── .dockerignore            # Keep Docker build context small
├── .env.example             # Template for env vars
├── .gitignore               # node_modules, dist, .env, *.log
├── commitlint.config.js     # Conventional commits config
├── eslint.config.js         # ESLint 9+ flat config with typescript-eslint
├── .prettierrc              # Prettier formatting rules
├── tsconfig.json            # TypeScript config
└── package.json             # Dependencies + scripts
```

---

## Monthly Cost

| Service                     | Cost              |
| --------------------------- | ----------------- |
| EC2 `t3.small` (2GB RAM)    | ~$15              |
| EC2 `t3.medium` (4GB RAM)   | ~$30              |
| EBS Storage (20GB SSD)      | ~$2               |
| Elastic IP (while attached) | Free              |
| **Total AWS**               | **~$17-32/month** |

### Free Services

| Service            | Free Tier                   |
| ------------------ | --------------------------- |
| Grafana Cloud Loki | 50GB/month logs             |
| UptimeRobot        | 50 monitors, 5-min interval |
| AWS CloudWatch     | Basic metrics + 10 alarms   |
| Let's Encrypt SSL  | Unlimited certificates      |
| GitHub Actions     | 2,000 min/month             |
| Docker Hub         | 1 private repo              |

---

## Troubleshooting

### App crashes at 3 AM:

```
UptimeRobot emails you → SSH into EC2 → docker compose ps → docker compose logs app -n 50 → Fix → Push → Build → Deploy
```

### .env missing after deploy:

The deploy workflow now backs up `.env` to `/tmp/` before cloning and restores it after. If it's still missing, recreate it manually: `nano ~/app/.env`

### SSH connection timeout on deploy:

Check EC2 Security Group → Ensure port 22 is open to `0.0.0.0/0` (or the GitHub runner's IP).

### Health check timeout on deploy:

Means the app container didn't become healthy in 120s. Check: `docker compose logs app --tail 50`

### Nginx not starting:

Usually means certbot dirs don't exist. Run: `mkdir -p ~/app/certbot/conf ~/app/certbot/www`
