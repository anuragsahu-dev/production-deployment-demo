# Production Architecture — Level 1

> **Single VPS Professional Deployment (0–50K Users)**
> AWS EC2 `t3.medium` • Docker Compose • Node.js + Express

---

## Architecture Overview

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
              │ ┌────────────┐ │        ┌──────────────────┐
         :80 ─┤►│   Nginx    │ │        │  AWS RDS         │
        :443 ─┤►│  (reverse  │ │        │  PostgreSQL 16   │
              │ │   proxy)   │ │        │  (managed DB)    │
              │ └─────┬──────┘ │        └────────▲─────────┘
              │       │        │                 │
              │       ▼        │                 │
              │ ┌────────────┐ │                 │
              │ │   App      ├─┼─────────────────┘
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

---

## What Each Component Does

| Component   | Purpose                                                         | RAM               | Docker Image           |
| ----------- | --------------------------------------------------------------- | ----------------- | ---------------------- |
| **Nginx**   | SSL termination, reverse proxy, rate limiting, security headers | ~10MB             | `nginx:alpine`         |
| **App**     | Node.js API (Express + Winston structured logging)              | ~200MB            | Custom `Dockerfile`    |
| **Redis**   | In-memory cache, rate limit counters, session store             | ~50-256MB         | `redis:7-alpine`       |
| **Alloy**   | Reads Docker logs → ships to Grafana Cloud Loki                 | ~50MB             | `grafana/alloy:v1.8.3` |
| **Certbot** | Obtains and renews Let's Encrypt SSL certificates               | Runs briefly      | `certbot/certbot`      |
| **Total**   |                                                                 | **~500MB of 4GB** |                        |

**Headroom:** 3.5GB free RAM — enough for 0-50K users comfortably.

---

## Deployment Strategy

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

### Git Hooks (Husky) — Quality Gates Before Git

| Hook         | When         | What Runs                               | Blocks If                                       |
| ------------ | ------------ | --------------------------------------- | ----------------------------------------------- |
| `pre-commit` | `git commit` | `npm run lint` + `npm run format:check` | Lint errors or bad formatting                   |
| `commit-msg` | `git commit` | `commitlint`                            | Not conventional format (`feat:`, `fix:`, etc.) |
| `pre-push`   | `git push`   | `npm run test` + `npm run build`        | Tests fail or build breaks                      |

### Branch Protection (GitHub Ruleset)

| Rule                               | Purpose                       |
| ---------------------------------- | ----------------------------- |
| Require pull request               | No direct push to main        |
| Require 1 approval (or 0 for solo) | Code review before merge      |
| Require status checks              | CI must pass before merge     |
| Require up-to-date branch          | PR tested against latest main |
| Block force pushes                 | Prevent history rewrite       |
| Restrict deletions                 | Can't delete main branch      |

---

## Monitoring Strategy

### The 3 Tools (All Free)

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  📡 UptimeRobot        → "Is the app alive?"           │
│     Pings /health every 5 min                           │
│     Emails you if 3 checks fail                         │
│     Free: 50 monitors                                   │
│                                                         │
│  📋 Grafana Cloud Loki  → "What went wrong?"            │
│     Alloy ships Docker logs to Loki                     │
│     Search, filter, and alert on logs                   │
│     Free: 50GB/month                                    │
│                                                         │
│  📊 AWS CloudWatch      → "Is the server healthy?"      │
│     Built into EC2 — no setup needed                    │
│     CPU, network, disk I/O, status checks               │
│     Free: basic metrics + 10 alarms                     │
│                                                         │
│  Total cost: $0/month                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Why These 3 (And Not More)

| ❌ Don't Need Yet                  | Why                                                            |
| ---------------------------------- | -------------------------------------------------------------- |
| Container metrics (cAdvisor)       | 5 containers on 1 server — SSH + `docker compose ps` is enough |
| Prometheus + Grafana (self-hosted) | Eats 2-3GB RAM on your 4GB server                              |
| Datadog / New Relic                | $15-50/month per host — overkill for pre-revenue               |
| Sentry                             | Add when you have paying users                                 |

### When Something Goes Wrong

```
3:00 AM — App crashes
    │
    ▼
UptimeRobot: 3 failed pings → emails you
    │
    ▼
You wake up, SSH into EC2:
    ssh -i key.pem ubuntu@<IP>
    │
    ▼
Quick check:
    docker compose ps              → Which container is down?
    docker compose logs app -n 50  → What happened?
    │
    ▼
Deep search (Grafana Cloud):
    {service="api"} | json | level="error"
    → Find the exact error with timestamp and context
    │
    ▼
Fix → Push → Build → Deploy
```

---

## Logging Strategy

### 3 Layers of Log Management

```
┌────────────────────────────────────────────────────┐
│  Layer 3: SEARCH & ALERTS (Grafana Cloud Loki)     │  ← Search logs, set alerts
│  Retention: 30 days (free tier)                    │
├────────────────────────────────────────────────────┤
│  Layer 2: LOG SHIPPING (Grafana Alloy)             │  ← Ships logs to cloud
│  Reads Docker logs → forwards to Loki             │
├────────────────────────────────────────────────────┤
│  Layer 1: LOCAL PROTECTION (Docker log rotation)   │  ← Prevents disk full
│  max-size: 10MB, max-file: 3, per container        │
└────────────────────────────────────────────────────┘
```

### The Log Flow

```
App (Winston)  →  console.log/info/error
       │
       ▼
Docker captures → /var/lib/docker/containers/xxx-json.log
       │                    │
       │                    ▼
       │            Docker log rotation
       │            (10MB × 3 files = 30MB max per container)
       │
       ▼
Alloy reads log files → parses JSON → adds labels
       │
       ▼
Grafana Cloud Loki (30 days retention, searchable)
```

### What to Log

| Level   | When                   | Example                                              |
| ------- | ---------------------- | ---------------------------------------------------- |
| `error` | Something broke        | `logger.error("Payment failed", { orderId, error })` |
| `warn`  | Something concerning   | `logger.warn("Rate limit hit", { ip })`              |
| `info`  | Business events        | `logger.info("User registered", { userId })`         |
| `debug` | Dev details (dev only) | `logger.debug("Cache miss", { key })`                |

### What NOT to Log

| ❌ Never Log          | Why                 |
| --------------------- | ------------------- |
| Passwords / tokens    | Security            |
| Credit card numbers   | PCI compliance      |
| Full request bodies   | May contain PII     |
| Health check requests | Noise — floods logs |

### Useful Loki Queries

```logql
# All app errors
{job="docker", service="api"} | json | level="error"

# Errors with stack traces
{job="docker", service="api"} | json | level="error" | line_format "{{.message}} {{.stack}}"

# Nginx 5xx errors
{job="docker", container_name="nginx"} |= "\" 5"

# Specific user activity
{job="docker", service="api"} | json | userId="user_123"

# Error count over time (for dashboards)
count_over_time({job="docker", service="api"} | json | level="error" [5m])
```

### Docker Log Rotation (Configure on EC2)

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
sudo systemctl restart docker
```

Without this, Docker logs grow forever and fill the disk.

---

## Security Checklist

### Network Security

| Rule              | Details                                           |
| ----------------- | ------------------------------------------------- |
| SSH (port 22)     | ⚠️ Restrict to **your IP only** — never 0.0.0.0/0 |
| HTTP (port 80)    | Open — Nginx handles, redirects to HTTPS          |
| HTTPS (port 443)  | Open — Nginx terminates SSL                       |
| App (port 3000)   | ❌ NOT exposed — internal only via Nginx          |
| Redis (port 6379) | ❌ NOT exposed — internal only via Docker network |
| RDS (port 5432)   | ❌ Only EC2 security group can access             |

### Application Security

| What               | How                                     |
| ------------------ | --------------------------------------- |
| Security headers   | Helmet in Express + Nginx headers       |
| Rate limiting      | Nginx: 10 req/s general, 5 req/min auth |
| CORS               | Restricted to frontend origin           |
| Body size limit    | Express: 10KB max JSON body             |
| Non-root container | `USER appuser` in Dockerfile            |
| Secrets management | `.env` on server only, never in Git     |
| SSL                | Let's Encrypt via Certbot, auto-renewal |

---

## Cost Breakdown

### Monthly

| Service                        | Cost           |
| ------------------------------ | -------------- |
| EC2 `t3.medium` (4GB RAM)      | ~$30           |
| RDS `db.t3.micro` (PostgreSQL) | ~$15           |
| EBS Storage (20GB SSD)         | ~$2            |
| Elastic IP (while attached)    | Free           |
| **Total AWS**                  | **~$47/month** |

### Free Services

| Service            | Free Tier                   |
| ------------------ | --------------------------- |
| Grafana Cloud Loki | 50GB/month logs             |
| UptimeRobot        | 50 monitors, 5-min interval |
| AWS CloudWatch     | Basic metrics + 10 alarms   |
| Let's Encrypt SSL  | Unlimited certs             |
| GitHub Actions     | 2,000 min/month             |
| Docker Hub         | 1 private repo              |

---

## Project Files

```
├── .github/workflows/
│   ├── ci.yaml              # PR checks: lint, build, test
│   ├── build.yaml           # Build Docker image → push to Hub
│   └── deploy.yaml          # SSH to EC2 → pull → restart
│
├── .husky/
│   ├── pre-commit           # lint + format check
│   ├── commit-msg           # commitlint (conventional commits)
│   └── pre-push             # test + build
│
├── alloy/
│   └── config.alloy         # Grafana Alloy: Docker logs → Loki
│
├── nginx/
│   ├── nginx.conf           # Main config: gzip, rate limit zones
│   └── conf.d/default.conf  # Server block: proxy, SSL, health
│
├── src/
│   ├── config/
│   │   ├── env.ts           # Centralized env vars
│   │   ├── logger.ts        # Winston: JSON prod, color dev
│   │   └── redis.ts         # ioredis client with reconnect
│   ├── index.ts             # Express app + graceful shutdown
│   ├── routes/              # API route handlers
│   ├── services/            # Business logic
│   └── data/                # Data layer
│
├── tests/                   # Vitest test files
│
├── compose.yaml             # Production: App + Redis + Nginx + Certbot + Alloy
├── compose.dev.yaml         # Development: App + Redis (hot reload)
├── Dockerfile               # Multi-stage production build
├── Dockerfile.dev           # Development with hot reload
├── .dockerignore             # Keep image small
├── .env.example             # Template for env vars
├── .gitignore               # Ignore node_modules, dist, .env, logs
├── commitlint.config.js     # Conventional commits config
├── tsconfig.json            # TypeScript config
├── package.json             # Dependencies + scripts
├── manual-steps.md          # Step-by-step server setup guide
└── architecture.md          # This file
```

---

## When to Move to Level 2

You'll know it's time when:

| Signal                     | Meaning                          |
| -------------------------- | -------------------------------- |
| CPU consistently > 70%     | Server is overloaded             |
| RAM consistently > 3GB     | Running out of memory            |
| Response times > 500ms avg | Too slow for users               |
| 50K+ active users          | Outgrowing single server         |
| Need zero-downtime deploys | Current setup has brief downtime |

**Level 2 adds:** Load balancer, multiple EC2 instances, container orchestration (ECS or K8s), CDN, auto-scaling.

But that's a problem for later. Level 1 handles 0-50K users on a single $47/month server.
