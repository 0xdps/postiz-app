# Postiz Self-Hosting Guide

> **Version:** v1.47.0 | **Last Updated:** 2026-06-07

This guide covers everything you need to self-host [Postiz](https://postiz.com) — an open-source social media scheduling platform supporting 15+ networks including X (Twitter), LinkedIn, Instagram, Facebook, YouTube, TikTok, Reddit, Pinterest, Threads, Bluesky, Mastodon, Discord, Slack, and more.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [System Requirements](#system-requirements)
- [Quick Start (Docker Compose)](#quick-start-docker-compose)
- [Configuration Reference](#configuration-reference)
  - [Required Settings](#required-settings)
  - [Storage Settings](#storage-settings)
  - [Social Media API Keys](#social-media-api-keys)
  - [Email Configuration](#email-configuration)
  - [OAuth / SSO](#oauth--sso)
  - [Payments (Stripe)](#payments-stripe)
  - [Short Link Services](#short-link-services)
- [Docker Compose Breakdown](#docker-compose-breakdown)
- [Manual Installation (Advanced)](#manual-installation-advanced)
- [Reverse Proxy Setup](#reverse-proxy-setup)
- [SSL / HTTPS](#ssl--https)
- [Backup & Restore](#backup--restore)
- [Updating Postiz](#updating-postiz)
- [Troubleshooting](#troubleshooting)
- [Environment Variable Reference](#environment-variable-reference)

---

## Architecture Overview

Postiz is a monorepo application consisting of multiple services:

| Service | Technology | Port | Purpose |
|---------|-----------|------|---------|
| **Frontend** | Next.js (React) | 4200 | Web UI |
| **Backend** | NestJS | 3000 | REST API |
| **Orchestrator** | NestJS + Temporal | 3002 | Background jobs & workflows |
| **PostgreSQL** | Postgres 17 | 5432 | Main database |
| **Redis** | Redis 7 | 6379 | Caching & queues |
| **Temporal** | Temporal Server | 7233 | Workflow engine |
| **Temporal UI** | Temporal UI | 8080 | Workflow monitoring |

The Docker image (`ghcr.io/gitroomhq/postiz-app:latest`) bundles the frontend, backend, and orchestrator behind an Nginx reverse proxy on port 5000.

```
┌─────────────────────────────────────────────────────────────┐
│                        Nginx (Port 5000)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   Frontend   │  │   Backend    │  │   Orchestrator   │   │
│  │   (Next.js)  │  │   (NestJS)   │  │   (Temporal)     │   │
│  │   :4200      │  │   :3000      │  │   :3002          │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         │                    │                   │
    ┌────┴────┐          ┌────┴────┐         ┌────┴────┐
    │  Redis  │          │ Postgres│         │Temporal │
    │  :6379  │          │  :5432  │         │  :7233  │
    └─────────┘          └─────────┘         └─────────┘
```

---

## System Requirements

### Minimum Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4+ cores |
| RAM | 4 GB | 8+ GB |
| Disk | 20 GB SSD | 50+ GB SSD |
| OS | Linux/macOS/Windows (WSL2) | Ubuntu 22.04 LTS |

### Network Requirements

- Port `4007` (or your chosen port) exposed for the web UI
- Port `8080` for Temporal UI (optional, internal only recommended)
- Outbound HTTPS (443) for social media APIs

### Software Requirements

- [Docker](https://docs.docker.com/get-docker/) 24.0+
- [Docker Compose](https://docs.docker.com/compose/install/) v2.0+
- (Optional) [Git](https://git-scm.com/downloads) for cloning

---

## Quick Start (Docker Compose)

The fastest way to get Postiz running is with the provided `docker-compose.yaml`.

### 1. Create a Project Directory

```bash
mkdir postiz && cd postiz
```

### 2. Download the Docker Compose File

```bash
curl -O https://raw.githubusercontent.com/gitroomhq/postiz-app/main/docker-compose.yaml
```

Or create it manually (see [Docker Compose Breakdown](#docker-compose-breakdown)).

### 3. Create the Dynamic Config Directory

```bash
mkdir -p dynamicconfig
cat > dynamicconfig/development-sql.yaml << 'EOF'
limit.maxIDLength:
  - value: 255
    constraints: {}
system.forceSearchAttributesCacheRefreshOnRead:
  - value: true
    constraints: {}
EOF
```

### 4. Configure Environment Variables

Create a `.env` file in the same directory:

```bash
cat > .env << 'EOF'
# === Required Settings ===
MAIN_URL=http://localhost:4007
FRONTEND_URL=http://localhost:4007
NEXT_PUBLIC_BACKEND_URL=http://localhost:4007/api
JWT_SECRET=change-this-to-a-random-string-min-32-chars
DATABASE_URL=postgresql://postiz-user:postiz-password@postiz-postgres:5432/postiz-db-local
REDIS_URL=redis://postiz-redis:6379
BACKEND_INTERNAL_URL=http://localhost:3000
TEMPORAL_ADDRESS=temporal:7233
IS_GENERAL=true
DISABLE_REGISTRATION=false

# === Storage ===
STORAGE_PROVIDER=local
UPLOAD_DIRECTORY=/uploads
NEXT_PUBLIC_UPLOAD_DIRECTORY=/uploads

# === Social Media APIs (fill in the ones you need) ===
# See Configuration Reference below

# === Optional: Email ===
# RESEND_API_KEY=your-resend-key
# EMAIL_FROM_ADDRESS=noreply@yourdomain.com
# EMAIL_FROM_NAME=Postiz

# === Optional: OpenAI ===
# OPENAI_API_KEY=sk-...
EOF
```

> **Important:** Change `JWT_SECRET` to a long random string. This is used for signing authentication tokens.

### 5. Start Postiz

```bash
docker compose up -d
```

### 6. Verify Everything is Running

```bash
docker compose ps
```

You should see all containers in an `Up` state:
- `postiz`
- `postiz-postgres`
- `postiz-redis`
- `temporal`
- `temporal-postgresql`
- `temporal-elasticsearch`
- `temporal-ui`
- `spotlight`

### 7. Access Postiz

Open your browser to:
- **Postiz App:** http://localhost:4007
- **Temporal UI:** http://localhost:8080

### 8. Create Your First Account

1. Navigate to http://localhost:4007
2. Click "Get Started" or "Sign Up"
3. Create your admin account
4. Start adding social media integrations

> **Note:** If `RESEND_API_KEY` is not configured, users are activated automatically. If configured, email verification is required.

---

## Configuration Reference

### Required Settings

These environment variables **must** be set for Postiz to function:

| Variable | Description | Example |
|----------|-------------|---------|
| `MAIN_URL` | The public URL users access Postiz on | `https://postiz.yourdomain.com` |
| `FRONTEND_URL` | Same as MAIN_URL in most cases | `https://postiz.yourdomain.com` |
| `NEXT_PUBLIC_BACKEND_URL` | Public API URL (usually `/api` path) | `https://postiz.yourdomain.com/api` |
| `JWT_SECRET` | Random secret for signing JWTs | `super-random-string-32-chars-min` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@host:5432/db` |
| `REDIS_URL` | Redis connection string | `redis://host:6379` |
| `BACKEND_INTERNAL_URL` | Internal backend URL | `http://localhost:3000` |
| `TEMPORAL_ADDRESS` | Temporal server address | `temporal:7233` |
| `IS_GENERAL` | Required flag | `true` |

### Storage Settings

Postiz supports two storage backends for uploaded media:

#### Option A: Local Storage (Default)

```env
STORAGE_PROVIDER=local
UPLOAD_DIRECTORY=/uploads
NEXT_PUBLIC_UPLOAD_DIRECTORY=/uploads
```

With Docker Compose, uploads are persisted in the `postiz-uploads` volume.

#### Option B: Cloudflare R2 (Recommended for Production)

```env
STORAGE_PROVIDER=cloudflare
CLOUDFLARE_ACCOUNT_ID=your-account-id
CLOUDFLARE_ACCESS_KEY=your-access-key
CLOUDFLARE_SECRET_ACCESS_KEY=your-secret-access-key
CLOUDFLARE_BUCKETNAME=your-bucket-name
CLOUDFLARE_BUCKET_URL=https://your-bucket-url.r2.cloudflarestorage.com/
CLOUDFLARE_REGION=auto
```

> Cloudflare R2 is currently required for saving social media avatars and profile pictures.

### Social Media API Keys

To enable posting to social platforms, you need to register OAuth applications with each provider:

| Platform | Variable Prefix | Setup Guide |
|----------|----------------|-------------|
| **X (Twitter)** | `X_API_KEY`, `X_API_SECRET` | [Twitter Developer Portal](https://developer.twitter.com) |
| **LinkedIn** | `LINKEDIN_CLIENT_ID`, `LINKEDIN_CLIENT_SECRET` | [LinkedIn Developers](https://developer.linkedin.com) |
| **Facebook** | `FACEBOOK_APP_ID`, `FACEBOOK_APP_SECRET` | [Meta for Developers](https://developers.facebook.com) |
| **Instagram** | Same as Facebook | Uses Facebook Graph API |
| **YouTube** | `YOUTUBE_CLIENT_ID`, `YOUTUBE_CLIENT_SECRET` | [Google Cloud Console](https://console.cloud.google.com) |
| **TikTok** | `TIKTOK_CLIENT_ID`, `TIKTOK_CLIENT_SECRET` | [TikTok for Developers](https://developers.tiktok.com) |
| **Reddit** | `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET` | [Reddit Apps](https://www.reddit.com/prefs/apps) |
| **Pinterest** | `PINTEREST_CLIENT_ID`, `PINTEREST_CLIENT_SECRET` | [Pinterest Developers](https://developers.pinterest.com) |
| **Threads** | `THREADS_APP_ID`, `THREADS_APP_SECRET` | [Meta for Developers](https://developers.facebook.com) |
| **GitHub** | `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` | [GitHub Settings > Developer](https://github.com/settings/developers) |
| **Discord** | `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, `DISCORD_BOT_TOKEN_ID` | [Discord Developer Portal](https://discord.com/developers) |
| **Slack** | `SLACK_ID`, `SLACK_SECRET`, `SLACK_SIGNING_SECRET` | [Slack API](https://api.slack.com) |
| **Mastodon** | `MASTODON_CLIENT_ID`, `MASTODON_CLIENT_SECRET` | Your instance settings |
| **Bluesky** | Built-in | No API keys needed |
| **Dribbble** | `DRIBBBLE_CLIENT_ID`, `DRIBBBLE_CLIENT_SECRET` | [Dribbble Developers](https://developer.dribbble.com) |

**Example configuration for X (Twitter):**

```env
X_URL=https://api.twitter.com
X_API_KEY=your-consumer-key
X_API_SECRET=your-consumer-secret
```

> For each OAuth app, set the callback URL to: `https://yourdomain.com/api/integrations/social/[provider]`

### Email Configuration

Email is used for user activation and notifications.

```env
RESEND_API_KEY=re_xxxxxxxx
EMAIL_FROM_ADDRESS=noreply@yourdomain.com
EMAIL_FROM_NAME=Postiz
```

- Get a Resend API key at [resend.com](https://resend.com)
- If `RESEND_API_KEY` is **not** set, users are activated automatically (no email verification)
- If `RESEND_API_KEY` **is** set, new users must verify their email before logging in

### OAuth / SSO

Postiz supports generic OAuth2 providers like Authentik, Keycloak, or Authelia:

```env
POSTIZ_GENERIC_OAUTH=true
POSTIZ_OAUTH_URL=https://auth.yourdomain.com
POSTIZ_OAUTH_AUTH_URL=https://auth.yourdomain.com/application/o/authorize
POSTIZ_OAUTH_TOKEN_URL=https://auth.yourdomain.com/application/o/token
POSTIZ_OAUTH_USERINFO_URL=https://auth.yourdomain.com/application/o/userinfo
POSTIZ_OAUTH_CLIENT_ID=your-client-id
POSTIZ_OAUTH_CLIENT_SECRET=your-client-secret
POSTIZ_OAUTH_SCOPE=openid profile email

NEXT_PUBLIC_POSTIZ_OAUTH_DISPLAY_NAME=Authentik
NEXT_PUBLIC_POSTIZ_OAUTH_LOGO_URL=https://your-logo-url.png
```

### Payments (Stripe)

For billing and subscriptions:

```env
FEE_AMOUNT=0.05
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_SECRET_KEY=sk_live_...
STRIPE_SIGNING_KEY=whsec_...
STRIPE_SIGNING_KEY_CONNECT=whsec_...
```

### Short Link Services

Optional integrations for link shortening:

```env
# Dub.co
DUB_TOKEN=your-dub-token
DUB_API_ENDPOINT=https://api.dub.co
DUB_SHORT_LINK_DOMAIN=dub.sh

# Short.io
SHORT_IO_SECRET_KEY=your-secret

# Kutt.it
KUTT_API_KEY=your-kutt-key
KUTT_API_ENDPOINT=https://kutt.it/api/v2
KUTT_SHORT_LINK_DOMAIN=kutt.it

# LinkDrip
LINK_DRIP_API_KEY=your-key
LINK_DRIP_API_ENDPOINT=https://api.linkdrip.com/v1/
LINK_DRIP_SHORT_LINK_DOMAIN=dripl.ink
```

---

## Docker Compose Breakdown

Here is the complete production `docker-compose.yaml` with explanations:

```yaml
services:
  # Main Postiz Application
  postiz:
    image: ghcr.io/gitroomhq/postiz-app:latest
    container_name: postiz
    restart: always
    environment:
      # Copy all variables from your .env file here
      MAIN_URL: 'https://postiz.yourdomain.com'
      FRONTEND_URL: 'https://postiz.yourdomain.com'
      NEXT_PUBLIC_BACKEND_URL: 'https://postiz.yourdomain.com/api'
      JWT_SECRET: 'your-random-secret-here'
      DATABASE_URL: 'postgresql://postiz-user:postiz-password@postiz-postgres:5432/postiz-db-local'
      REDIS_URL: 'redis://postiz-redis:6379'
      BACKEND_INTERNAL_URL: 'http://localhost:3000'
      TEMPORAL_ADDRESS: "temporal:7233"
      IS_GENERAL: 'true'
      DISABLE_REGISTRATION: 'false'
      STORAGE_PROVIDER: 'local'
      UPLOAD_DIRECTORY: '/uploads'
      NEXT_PUBLIC_UPLOAD_DIRECTORY: '/uploads'
    volumes:
      - postiz-config:/config/
      - postiz-uploads:/uploads/
    ports:
      - "4007:5000"
    networks:
      - postiz-network
      - temporal-network
    depends_on:
      postiz-postgres:
        condition: service_healthy
      postiz-redis:
        condition: service_healthy

  # PostgreSQL Database
  postiz-postgres:
    image: postgres:17-alpine
    container_name: postiz-postgres
    restart: always
    environment:
      POSTGRES_PASSWORD: postiz-password
      POSTGRES_USER: postiz-user
      POSTGRES_DB: postiz-db-local
    volumes:
      - postgres-volume:/var/lib/postgresql/data
    networks:
      - postiz-network
    healthcheck:
      test: pg_isready -U postiz-user -d postiz-db-local
      interval: 10s
      timeout: 3s
      retries: 3

  # Redis Cache/Queue
  postiz-redis:
    image: redis:7.2
    container_name: postiz-redis
    restart: always
    healthcheck:
      test: redis-cli ping
      interval: 10s
      timeout: 3s
      retries: 3
    volumes:
      - postiz-redis-data:/data
    networks:
      - postiz-network

  # Sentry Spotlight (Debugging)
  spotlight:
    pull_policy: always
    container_name: spotlight
    ports:
      - 127.0.0.1:8969:8969/tcp  # Bind to localhost only
    image: ghcr.io/getsentry/spotlight:latest
    networks:
      - postiz-network

  # Temporal Elasticsearch
  temporal-elasticsearch:
    container_name: temporal-elasticsearch
    image: elasticsearch:7.17.27
    environment:
      - cluster.routing.allocation.disk.threshold_enabled=true
      - cluster.routing.allocation.disk.watermark.low=512mb
      - cluster.routing.allocation.disk.watermark.high=256mb
      - cluster.routing.allocation.disk.watermark.flood_stage=128mb
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms256m -Xmx256m
      - xpack.security.enabled=false
    networks:
      - temporal-network
    expose:
      - 9200
    volumes:
      - temporal-es-data:/var/lib/elasticsearch/data

  # Temporal PostgreSQL
  temporal-postgresql:
    container_name: temporal-postgresql
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: temporal
      POSTGRES_USER: temporal
    networks:
      - temporal-network
    expose:
      - 5432
    volumes:
      - temporal-postgres-data:/var/lib/postgresql/data

  # Temporal Server
  temporal:
    container_name: temporal
    ports:
      - '127.0.0.1:7233:7233'  # Bind to localhost only
    image: temporalio/auto-setup:1.28.1
    depends_on:
      - temporal-postgresql
      - temporal-elasticsearch
    environment:
      - DB=postgres12
      - DB_PORT=5432
      - POSTGRES_USER=temporal
      - POSTGRES_PWD=temporal
      - POSTGRES_SEEDS=temporal-postgresql
      - DYNAMIC_CONFIG_FILE_PATH=config/dynamicconfig/development-sql.yaml
      - ENABLE_ES=true
      - ES_SEEDS=temporal-elasticsearch
      - ES_VERSION=v7
      - TEMPORAL_NAMESPACE=default
    networks:
      - temporal-network
    volumes:
      - ./dynamicconfig:/etc/temporal/config/dynamicconfig

  # Temporal Admin Tools
  temporal-admin-tools:
    container_name: temporal-admin-tools
    image: temporalio/admin-tools:1.28.1-tctl-1.18.4-cli-1.4.1
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CLI_ADDRESS=temporal:7233
    networks:
      - temporal-network
    stdin_open: true
    depends_on:
      - temporal
    tty: true

  # Temporal Web UI
  temporal-ui:
    container_name: temporal-ui
    image: temporalio/ui:2.34.0
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CORS_ORIGINS=http://127.0.0.1:3000
    networks:
      - temporal-network
    ports:
      - "127.0.0.1:8080:8080"  # Bind to localhost only

volumes:
  postgres-volume:
  postiz-redis-data:
  postiz-config:
  postiz-uploads:
  temporal-es-data:
  temporal-postgres-data:

networks:
  postiz-network:
  temporal-network:
    driver: bridge
    name: temporal-network
```

### Security Hardening

For production, bind internal services to localhost:

```yaml
ports:
  - "127.0.0.1:8080:8080"   # Temporal UI
  - "127.0.0.1:7233:7233"   # Temporal gRPC
  - "127.0.0.1:8969:8969"   # Spotlight
```

Only expose Postiz (port 4007) to the public internet.

---

## Manual Installation (Advanced)

If you prefer not to use Docker, you can run Postiz from source.

### Prerequisites

- Node.js 22.12.0+ (but < 23.0.0)
- pnpm 10.6.1+
- PostgreSQL 17
- Redis 7
- Temporal Server

### 1. Clone the Repository

```bash
git clone https://github.com/gitroomhq/postiz-app.git
cd postiz-app
```

### 2. Install Dependencies

```bash
pnpm install
```

### 3. Set Up Environment

```bash
cp .env.example .env
# Edit .env with your configuration
```

### 4. Set Up the Database

```bash
# Push the Prisma schema to your database
pnpm run prisma-db-push
```

### 5. Start Temporal Server

Follow the [Temporal Server installation guide](https://docs.temporal.io/self-hosted-guide/setup).

### 6. Start All Services

In separate terminal windows:

```bash
# Terminal 1: Backend
pnpm run dev:backend

# Terminal 2: Frontend
pnpm run dev:frontend

# Terminal 3: Orchestrator
pnpm run dev:orchestrator
```

Or use the combined dev command:

```bash
pnpm run dev
```

### 7. Production Build

```bash
# Build all apps
pnpm run build

# Start with PM2
pnpm run pm2
```

---

## Reverse Proxy Setup

For production, place Postiz behind a reverse proxy like Nginx, Caddy, or Traefik.

### Nginx Example

```nginx
server {
    listen 80;
    server_name postiz.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name postiz.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    client_max_body_size 2G;

    location / {
        proxy_pass http://localhost:4007;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Caddy Example

```caddy
postiz.yourdomain.com {
    reverse_proxy localhost:4007
}
```

### Important Headers

Postiz relies on these headers being passed through:

- `Host`
- `X-Real-IP`
- `X-Forwarded-For`
- `X-Forwarded-Proto`
- `Accept-Language`

---

## SSL / HTTPS

Postiz requires HTTPS for OAuth callbacks from most social media platforms.

### Using Let's Encrypt with Caddy

Caddy automatically handles SSL:

```bash
caddy reverse-proxy --from postiz.yourdomain.com --to localhost:4007
```

### Using Let's Encrypt with Certbot + Nginx

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d postiz.yourdomain.com
```

### Cloudflare Tunnel (Easiest)

If using Cloudflare, you can use a tunnel:

```bash
cloudflared tunnel --url http://localhost:4007
```

---

## Backup & Restore

### Backup

```bash
# Backup PostgreSQL
docker exec postiz-postgres pg_dump -U postiz-user postiz-db-local > postiz-backup-$(date +%Y%m%d).sql

# Backup uploads
docker run --rm -v postiz-uploads:/uploads -v $(pwd):/backup alpine tar czf /backup/postiz-uploads-$(date +%Y%m%d).tar.gz -C /uploads .
```

### Restore

```bash
# Restore PostgreSQL
docker exec -i postiz-postgres psql -U postiz-user -d postiz-db-local < postiz-backup-YYYYMMDD.sql

# Restore uploads
docker run --rm -v postiz-uploads:/uploads -v $(pwd):/backup alpine sh -c "cd /uploads && tar xzf /backup/postiz-uploads-YYYYMMDD.tar.gz"
```

### Automated Backups

Add to crontab (`crontab -e`):

```bash
# Daily backup at 2 AM
0 2 * * * cd /path/to/postiz && docker exec postiz-postgres pg_dump -U postiz-user postiz-db-local > backups/postiz-$(date +\%Y\%m\%d).sql 2>/dev/null
0 2 * * * find /path/to/postiz/backups -name "postiz-*.sql" -mtime +7 -delete
```

---

## Updating Postiz

### Docker Compose Update

```bash
cd /path/to/postiz

# Pull the latest image
docker compose pull

# Restart with the new image
docker compose up -d

# Clean up old images
docker image prune -f
```

### Check the Current Version

```bash
docker exec postiz cat /app/version.txt
```

### View Logs After Update

```bash
docker compose logs -f postiz
```

---

## Building Your Own Docker Image

The official Docker image is built automatically via GitHub Actions (see `.github/workflows/build-containers.yml`). You can also build your own image from source.

### Using the GitHub Actions Workflow

The repository includes a workflow that builds and pushes multi-arch (`amd64` + `arm64`) images to GitHub Container Registry (ghcr.io).

**Triggers:**
- **Tags** — Pushing a tag like `v1.2.3` creates `ghcr.io/gitroomhq/postiz-app:v1.2.3` and updates `latest`
- **Main branch** — Pushes to `main` create `ghcr.io/gitroomhq/postiz-app:dev-<sha>`
- **Pull requests** — Builds the image but does not push (validation only)
- **Manual** — Can be triggered from the Actions tab

**Features:**
- Multi-architecture support (AMD64 + ARM64)
- Build caching via GitHub Actions cache
- Automatic vulnerability scanning with Trivy
- Artifact attestations for supply chain security
- Build summary in the GitHub Actions UI

### Building Locally

```bash
# Clone the repository
git clone https://github.com/gitroomhq/postiz-app.git
cd postiz-app

# Build the image
docker build -f Dockerfile.dev -t postiz:local .

# Run locally
docker run -p 4007:5000 postiz:local
```

### Custom Registry

To push to a different registry, modify the workflow or build manually:

```bash
# Build and tag for your registry
docker build -f Dockerfile.dev -t your-registry.com/postiz:custom .

# Push
docker push your-registry.com/postiz:custom
```

---

## Troubleshooting

### Container Won't Start

```bash
# Check logs
docker compose logs postiz

# Check for port conflicts
sudo lsof -i :4007
sudo lsof -i :5432
sudo lsof -i :6379
```

### Database Connection Issues

```bash
# Test PostgreSQL connection
docker exec -it postiz-postgres psql -U postiz-user -d postiz-db-local -c "\dt"

# Check if database is initialized
docker exec postiz-postgres pg_isready -U postiz-user
```

### Redis Connection Issues

```bash
# Test Redis
docker exec -it postiz-redis redis-cli ping
# Should return: PONG
```

### Temporal Issues

```bash
# Check Temporal server status
docker compose logs temporal

# Check Temporal UI
curl http://localhost:8080/api/v1/namespaces
```

### OAuth Callback Failures

1. Ensure `MAIN_URL` and `FRONTEND_URL` use HTTPS in production
2. Verify callback URLs in each social platform's app settings
3. Check that the backend is accessible at `NEXT_PUBLIC_BACKEND_URL`

### Uploads Not Working

1. Check storage provider settings
2. Verify volume mounts: `docker volume ls | grep postiz`
3. Check disk space: `df -h`

### Reset Everything (⚠️ Destructive)

```bash
docker compose down -v  # Removes all volumes
docker compose up -d    # Starts fresh
```

---

## Environment Variable Reference

### Core Application

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `MAIN_URL` | Yes | - | Public URL of your Postiz instance |
| `FRONTEND_URL` | Yes | - | Same as MAIN_URL |
| `NEXT_PUBLIC_BACKEND_URL` | Yes | - | Public API endpoint |
| `BACKEND_INTERNAL_URL` | Yes | `http://localhost:3000` | Internal backend URL |
| `JWT_SECRET` | Yes | - | Secret for JWT signing |
| `DATABASE_URL` | Yes | - | PostgreSQL connection string |
| `REDIS_URL` | Yes | - | Redis connection string |
| `TEMPORAL_ADDRESS` | Yes | `temporal:7233` | Temporal server address |
| `IS_GENERAL` | Yes | `true` | Required flag |
| `DISABLE_REGISTRATION` | No | `false` | Set to `true` to disable new signups |
| `API_LIMIT` | No | `30` | Public API rate limit (requests/hour) |
| `NOT_SECURED` | No | - | Set for non-HTTPS dev environments |

### Storage

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `STORAGE_PROVIDER` | No | `local` | `local` or `cloudflare` |
| `UPLOAD_DIRECTORY` | No | - | Local upload path |
| `NEXT_PUBLIC_UPLOAD_DIRECTORY` | No | - | Public upload path |
| `CLOUDFLARE_ACCOUNT_ID` | No | - | R2 account ID |
| `CLOUDFLARE_ACCESS_KEY` | No | - | R2 access key |
| `CLOUDFLARE_SECRET_ACCESS_KEY` | No | - | R2 secret key |
| `CLOUDFLARE_BUCKETNAME` | No | - | R2 bucket name |
| `CLOUDFLARE_BUCKET_URL` | No | - | R2 public URL |
| `CLOUDFLARE_REGION` | No | `auto` | R2 region |

### Social Media APIs

| Variable | Description |
|----------|-------------|
| `X_URL` | X API base URL |
| `X_API_KEY` | X API key |
| `X_API_SECRET` | X API secret |
| `LINKEDIN_CLIENT_ID` | LinkedIn app ID |
| `LINKEDIN_CLIENT_SECRET` | LinkedIn app secret |
| `REDDIT_CLIENT_ID` | Reddit app ID |
| `REDDIT_CLIENT_SECRET` | Reddit app secret |
| `GITHUB_CLIENT_ID` | GitHub OAuth app ID |
| `GITHUB_CLIENT_SECRET` | GitHub OAuth secret |
| `THREADS_APP_ID` | Threads app ID |
| `THREADS_APP_SECRET` | Threads app secret |
| `FACEBOOK_APP_ID` | Facebook app ID |
| `FACEBOOK_APP_SECRET` | Facebook app secret |
| `YOUTUBE_CLIENT_ID` | YouTube client ID |
| `YOUTUBE_CLIENT_SECRET` | YouTube client secret |
| `TIKTOK_CLIENT_ID` | TikTok app ID |
| `TIKTOK_CLIENT_SECRET` | TikTok app secret |
| `PINTEREST_CLIENT_ID` | Pinterest app ID |
| `PINTEREST_CLIENT_SECRET` | Pinterest app secret |
| `DRIBBBLE_CLIENT_ID` | Dribbble app ID |
| `DRIBBBLE_CLIENT_SECRET` | Dribbble app secret |
| `DISCORD_CLIENT_ID` | Discord app ID |
| `DISCORD_CLIENT_SECRET` | Discord app secret |
| `DISCORD_BOT_TOKEN_ID` | Discord bot token |
| `SLACK_ID` | Slack app ID |
| `SLACK_SECRET` | Slack app secret |
| `SLACK_SIGNING_SECRET` | Slack signing secret |
| `MASTODON_URL` | Mastodon instance URL |
| `MASTODON_CLIENT_ID` | Mastodon app ID |
| `MASTODON_CLIENT_SECRET` | Mastodon app secret |

### Email

| Variable | Required | Description |
|----------|----------|-------------|
| `RESEND_API_KEY` | No | Resend API key for email |
| `EMAIL_FROM_ADDRESS` | No | Sender email address |
| `EMAIL_FROM_NAME` | No | Sender display name |

### OAuth / SSO

| Variable | Description |
|----------|-------------|
| `POSTIZ_GENERIC_OAUTH` | Enable generic OAuth (`true`/`false`) |
| `POSTIZ_OAUTH_URL` | OAuth provider base URL |
| `POSTIZ_OAUTH_AUTH_URL` | Authorization endpoint |
| `POSTIZ_OAUTH_TOKEN_URL` | Token endpoint |
| `POSTIZ_OAUTH_USERINFO_URL` | UserInfo endpoint |
| `POSTIZ_OAUTH_CLIENT_ID` | OAuth client ID |
| `POSTIZ_OAUTH_CLIENT_SECRET` | OAuth client secret |
| `POSTIZ_OAUTH_SCOPE` | OAuth scopes |
| `NEXT_PUBLIC_POSTIZ_OAUTH_DISPLAY_NAME` | Button label |
| `NEXT_PUBLIC_POSTIZ_OAUTH_LOGO_URL` | Provider logo URL |

### Payments

| Variable | Description |
|----------|-------------|
| `FEE_AMOUNT` | Platform fee (0.05 = 5%) |
| `STRIPE_PUBLISHABLE_KEY` | Stripe publishable key |
| `STRIPE_SECRET_KEY` | Stripe secret key |
| `STRIPE_SIGNING_KEY` | Stripe webhook signing secret |
| `STRIPE_SIGNING_KEY_CONNECT` | Stripe Connect webhook secret |

### AI / Misc

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key for AI features |
| `NEXT_PUBLIC_DISCORD_SUPPORT` | Discord support server URL |
| `NEXT_PUBLIC_POLOTNO` | Polotno API key |
| `EXTENSION_ID` | Chrome extension ID |

### Short Links

| Variable | Description |
|----------|-------------|
| `DUB_TOKEN` | Dub.co API token |
| `DUB_API_ENDPOINT` | Dub API endpoint |
| `DUB_SHORT_LINK_DOMAIN` | Dub domain |
| `SHORT_IO_SECRET_KEY` | Short.io secret |
| `KUTT_API_KEY` | Kutt API key |
| `KUTT_API_ENDPOINT` | Kutt endpoint |
| `KUTT_SHORT_LINK_DOMAIN` | Kutt domain |
| `LINK_DRIP_API_KEY` | LinkDrip API key |
| `LINK_DRIP_API_ENDPOINT` | LinkDrip endpoint |
| `LINK_DRIP_SHORT_LINK_DOMAIN` | LinkDrip domain |

### Sentry (Monitoring)

| Variable | Description |
|----------|-------------|
| `NEXT_PUBLIC_SENTRY_DSN` | Sentry DSN |
| `SENTRY_SPOTLIGHT` | Enable Spotlight debugging |

---

## Support & Resources

- **Documentation:** https://docs.postiz.com
- **Public API Docs:** https://docs.postiz.com/public-api
- **Discord (Devs):** https://discord.postiz.com
- **GitHub Issues:** https://github.com/gitroomhq/postiz-app/issues
- **YouTube Tutorials:** https://youtube.com/@postizofficial

---

## License

Postiz is licensed under [AGPL-3.0](https://opensource.org/license/agpl-v3).
