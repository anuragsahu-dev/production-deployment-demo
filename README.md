# Production-Grade Docker Compose Deployment

A production-ready deployment setup for a Node.js REST API on a single AWS EC2 instance (4 GB RAM).  
Covers reverse proxying, SSL, CI/CD, structured logging, monitoring, and container orchestration — all running for ~$47/month.

![Node.js](https://img.shields.io/badge/Node.js-22-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Docker](https://img.shields.io/badge/Docker_Compose-2496ED?style=flat-square&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=flat-square&logo=amazonec2&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana_Loki-F46800?style=flat-square&logo=grafana&logoColor=white)

---

## What This Project Demonstrates

This repository demonstrates how to deploy a Node.js backend to production using a cost-efficient single-server architecture.

It highlights:

- Dockerized backend deployment with multi-stage builds
- Reverse proxy configuration with Nginx (SSL, rate limiting, gzip)
- CI/CD automation using GitHub Actions (3 separate workflows)
- Observability with structured logging and log aggregation (Winston → Alloy → Grafana Loki)
- Basic monitoring and uptime checks (UptimeRobot, CloudWatch)
- Secure production configuration practices (non-root containers, secrets management, security headers)

---

## Architecture Overview

```
                    Internet
                       |
                       v
              +----------------+
              |   AWS EC2      |
              |   t3.medium    |
              |   4 GB RAM     |
              |                |
              | +------------+ |
         :80 -|>|   Nginx    | |
        :443 -|>| (reverse   | |
              | |  proxy +   | |
              | |  SSL +     | |
              | | rate limit)| |
              | +-----+------+ |
              |       |        |
              |       v        |
              | +------------+ |
              | |    App     | |
              | | (Node.js + | |
              | |  Express + | |
              | |  Winston)  | |
              | +-----+------+ |
              |       |        |
              |       v        |
              | +------------+ |
              | |   Redis    | |
              | | (cache +   | |
              | | rate limit)| |
              | +------------+ |
              |                |
              | +------------+ |      +-----------------------+
              | |   Alloy    |-|----->| Grafana Cloud Loki    |
              | | (log       | |      | (log search + alerts) |
              | |  shipper)  | |      +-----------------------+
              | +------------+ |
              |                |
              | +------------+ |
              | |  Certbot   | |
              | |  (SSL)     | |
              | +------------+ |
              +----------------+
                       |
              +--------+--------+     +-----------------+
              | AWS CloudWatch  |     |  UptimeRobot    |
              | (server metrics)|     |  (uptime check) |
              +-----------------+     +-----------------+
```

**Request flow:** Client → Nginx (SSL termination, rate limiting, gzip) → App (:3000, internal only) → Redis (cache) → Response

---

## Key Features

- **Multi-stage Docker build** — non-root user, pruned devDependencies, source maps enabled
- **Nginx reverse proxy** — SSL termination, gzip compression, rate limiting (10 req/s API, 5 req/min auth)
- **SSL/TLS** via Let's Encrypt with Certbot
- **3-workflow CI/CD** — automated PR checks, manual image build + push, manual SSH deploy with health checks
- **Structured logging** — Winston (JSON in prod) → Grafana Alloy → Grafana Cloud Loki
- **Monitoring** — UptimeRobot (uptime), CloudWatch (server metrics), Loki (log search + alerts)
- **Git quality gates** — Husky pre-commit/pre-push hooks, commitlint, ESLint, Prettier
- **Branch protection** — required PR, status checks, force-push blocked
- **Graceful shutdown** — handles SIGTERM/SIGINT, drains connections, closes Redis
- **Health checks** — on every container, used by deploy workflow for rollout verification

---

## Tech Stack

| Layer          | Technology                          | Purpose                                    |
| -------------- | ----------------------------------- | ------------------------------------------ |
| Runtime        | Node.js 22, TypeScript 5.9          | Application server with strict type safety |
| Framework      | Express 5                           | HTTP server (Helmet, CORS, body limits)    |
| Cache          | Redis 7 Alpine                      | In-memory cache with LRU eviction          |
| Reverse Proxy  | Nginx stable-alpine                 | SSL, rate limiting, gzip, security headers |
| SSL            | Certbot / Let's Encrypt             | Free TLS certificates, auto-renewal        |
| Logging        | Winston → Alloy → Loki              | Structured JSON logs to Grafana Cloud      |
| CI/CD          | GitHub Actions (3 workflows)        | Lint, test, build image, deploy via SSH    |
| Containers     | Docker, Docker Compose              | Multi-stage builds, health checks          |
| Infrastructure | AWS EC2 t3.medium                   | Single VPS, Elastic IP                     |
| Monitoring     | UptimeRobot, CloudWatch             | Uptime pings, server-level metrics         |
| Code Quality   | Husky, CommitLint, ESLint, Prettier | Git hooks, conventional commits            |
| Testing        | Vitest                              | Unit tests with CI service containers      |

---

## Deployment Architecture

### Container Layout (Single EC2, ~500 MB of 4 GB used)

| Container | Image                  | RAM     | Role                                    |
| --------- | ---------------------- | ------- | --------------------------------------- |
| Nginx     | `nginx:stable-alpine`  | ~10 MB  | Reverse proxy, SSL, rate limiting       |
| App       | Custom multi-stage     | ~200 MB | Node.js API, Winston structured logging |
| Redis     | `redis:7-alpine`       | ~50 MB  | Cache, LRU eviction (256 MB cap)        |
| Alloy     | `grafana/alloy:v1.8.3` | ~50 MB  | Ships Docker logs to Grafana Cloud Loki |
| Certbot   | `certbot/certbot`      | Brief   | Obtains and renews SSL certificates     |

### Network Security

| Port | Exposure       | Purpose                                   |
| ---- | -------------- | ----------------------------------------- |
| 22   | SSH (key-only) | Admin access                              |
| 80   | Public         | HTTP → redirects to HTTPS                 |
| 443  | Public         | HTTPS, Nginx terminates SSL               |
| 3000 | Internal       | App, accessible only via Docker network   |
| 6379 | Internal       | Redis, accessible only via Docker network |

---

## CI/CD Pipeline Overview

```
Feature Branch → PR → CI (auto) → Merge → Build (manual) → Deploy (manual)
```

| Workflow      | Trigger         | Steps                                                                                  |
| ------------- | --------------- | -------------------------------------------------------------------------------------- |
| `ci.yaml`     | PR to `main`    | Lint + Build (Node 20, 22, 24), Unit tests with Redis service                          |
| `build.yaml`  | Manual dispatch | Build Docker image, push to Docker Hub, update tag in compose.yaml                     |
| `deploy.yaml` | Manual dispatch | SSH to EC2, git pull, docker pull, `docker compose up -d`, health check (120s timeout) |

**Build and Deploy are manual** — gives full control over what goes live and when.

### Git Hooks (Husky)

| Hook         | Trigger      | Runs                                |
| ------------ | ------------ | ----------------------------------- |
| `pre-commit` | `git commit` | `lint` + `format:check`             |
| `commit-msg` | `git commit` | `commitlint` (conventional commits) |
| `pre-push`   | `git push`   | `test` + `build`                    |

---

## Project Structure

```
.github/workflows/
  ci.yaml                  # PR checks: lint, build, test (3 Node versions)
  build.yaml               # Build + push Docker image, update compose.yaml tag
  deploy.yaml              # SSH deploy to EC2 with health check

.husky/
  pre-commit               # lint + format check
  commit-msg               # commitlint
  pre-push                 # test + build

alloy/
  config.alloy             # Grafana Alloy: Docker logs → Loki

nginx/
  nginx.conf               # Gzip, rate limit zones, performance tuning
  conf.d/default.conf      # Reverse proxy, SSL, ACME challenge, health bypass

src/
  config/
    env.ts                 # Centralized env config
    logger.ts              # Winston (JSON prod, colorized dev)
    redis.ts               # ioredis with retry strategy
  data/                    # Static data layer
  routes/                  # Express route handlers
  services/                # Business logic
  index.ts                 # App entry, health endpoint, graceful shutdown

tests/                     # Vitest unit tests
docs/
  architecture.md          # Full architecture documentation
  deployment-notes.md      # Step-by-step deployment guide

compose.yaml               # Production (App + Redis + Nginx + Certbot + Alloy)
compose.dev.yaml           # Development (App + Redis, hot reload)
Dockerfile                 # Multi-stage production build, non-root user
Dockerfile.dev             # Dev build with tsx watch
```

---

## Local Development Setup

```bash
# Clone
git clone https://github.com/anuragsahu-dev/Production-Deploy.git
cd Production-Deploy

# Create env file
cp .env.example .env

# Start dev environment (hot reload)
docker compose -f compose.dev.yaml up --build

# Available at:
#   http://localhost:3000/health
#   http://localhost:3000/api/users
#   http://localhost:3000/api/products
```

```bash
# Tests
npm install
npm run test

# Linting + formatting
npm run lint
npm run format:check
```

---

## Production Deployment Summary

> Full step-by-step guide: [docs/deployment-notes.md](docs/deployment-notes.md)

1. **Provision** — Launch EC2 `t3.medium` (Ubuntu 24.04), attach Elastic IP, configure security group (ports 22/80/443 only)
2. **Setup server** — Install Docker via official apt repo, configure log rotation (`10m × 3` per container), create `.env` with production secrets
3. **DNS** — Point domain to Elastic IP (DuckDNS or any DNS provider)
4. **First deploy** — Run Build workflow → Run Deploy workflow → Verify health check
5. **SSL** — Run Certbot to obtain Let's Encrypt certificate, enable HTTPS in Nginx config, restart
6. **Monitoring** — Grafana Cloud Loki receives logs via Alloy, UptimeRobot pings `/health`, CloudWatch tracks server metrics
7. **Subsequent deploys** — Build workflow (new image) → Deploy workflow (pull + restart + health check)

---

## Cost Overview

| Item                         | Monthly Cost |
| ---------------------------- | ------------ |
| EC2 `t3.medium` (4 GB)       | ~$30         |
| RDS `db.t3.micro` (optional) | ~$15         |
| EBS Storage (20 GB gp3)      | ~$2          |
| **Total AWS**                | **~$47**     |

Grafana Cloud Loki (50 GB/mo), UptimeRobot (50 monitors), CloudWatch (basic), Let's Encrypt, GitHub Actions (2,000 min/mo), Docker Hub (1 private repo) — all on **free tiers**.

---

## Documentation

| Document                                             | Description                                                                        |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------- |
| [docs/architecture.md](docs/architecture.md)         | Architecture diagrams, monitoring strategy, security checklist, scaling signals    |
| [docs/deployment-notes.md](docs/deployment-notes.md) | Complete deployment walkthrough — EC2, Docker, SSL, CI/CD secrets, troubleshooting |

---

## Author

**Anurag Sahu** — [GitHub](https://github.com/anuragsahu-dev)

Built to demonstrate production deployment and DevOps practices for single-server setups.
