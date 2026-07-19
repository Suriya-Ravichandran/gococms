# Docker Architecture

> GOCO CMS is Docker-first: every environment — laptop, CI, staging, production — is spun up from the same image set and Compose stack, so "works on my machine" becomes "works in every workspace."

GOCO CMS ("The Open Source Website Operating System") ships as an OpenSwoole long-running application, not a stateless PHP-FPM worker behind a web server. That single fact shapes the entire container design: the app process **is** the HTTP server (ZealPHP on OpenSwoole 22.1+, PHP 8.4+), Traefik terminates TLS and routes to it, and the stateful backends (MongoDB, Redis, MinIO, Meilisearch) live in sibling containers connected over a private Docker network.

This document is the authoritative reference for the images, Compose files, services, networks, and volumes that make up a GOCO deployment. For the edge/proxy layer see [traefik.md](traefik.md); for the end-to-end rollout procedure see [deployment-guide.md](deployment-guide.md); for the environment variables referenced throughout, see [../getting-started/configuration.md](../getting-started/configuration.md).

**Stability:** `stable` (image + Compose contract). The optional Watchtower auto-update path is `beta`.

---

## Design Principles

- **One image, many modes.** The production image is built once and promoted through environments; only environment variables and Compose overlays change between staging and production.
- **The app is the server.** No Nginx or Apache in front of PHP. ZealPHP/OpenSwoole binds `0.0.0.0:8080` inside the container; Traefik is the only public listener.
- **Stateful data lives in named volumes**, never in the app container. The app container is disposable and horizontally scalable.
- **Graceful shutdown everywhere.** OpenSwoole drains in-flight coroutines on `SIGTERM`; every service declares a `stop_grace_period` and a healthcheck so orchestrators reschedule safely.
- **Overlay, don't fork.** A base `docker-compose.yml` holds the shared truth; `docker-compose.dev.yml` and `docker-compose.prod.yml` overlay only what differs.

```mermaid
flowchart TB
    subgraph edge["Public edge"]
        TR[Traefik<br/>:80 :443 UDP:443 HTTP/3]
    end
    subgraph app["App tier — scalable"]
        G1[gococms #1<br/>ZealPHP / OpenSwoole :8080]
        G2[gococms #2]
    end
    subgraph data["Stateful tier — private network"]
        MDB[(MongoDB)]
        RDS[(Redis)]
        MIN[(MinIO)]
        MEI[(Meilisearch)]
        MP[Mailpit]
    end
    Internet -->|443/HTTP3| TR
    TR -->|:8080| G1
    TR -->|:8080| G2
    G1 --- MDB & RDS & MIN & MEI & MP
    G2 --- MDB & RDS & MIN & MEI & MP
    WT[Watchtower<br/>optional] -.watches.-> G1
```

---

## Images

GOCO builds three artifacts: a lean **production** image, a fatter **dev** image with hot reload and tooling, and a `.dockerignore` that keeps both small and reproducible.

### `Dockerfile` (production, multi-stage)

The production image is a three-stage build:

1. **`vendor`** — install Composer dependencies with a warm cache and no dev packages.
2. **`runtime-base`** — a PHP 8.4 CLI base with OpenSwoole 22.1+ and the `ext-zealphp` extension compiled in.
3. **`app`** — copy the vendored deps + application source onto the runtime base, drop privileges, expose `8080`, and launch `app.php`.

```dockerfile
# syntax=docker/dockerfile:1.7

# ---------- Stage 1: Composer dependencies ----------
FROM composer:2 AS vendor
WORKDIR /build
COPY composer.json composer.lock ./
RUN --mount=type=cache,target=/tmp/composer-cache \
    composer install \
      --no-dev --no-scripts --no-autoloader \
      --prefer-dist --no-interaction
COPY . .
RUN composer dump-autoload --optimize --classmap-authoritative --no-dev

# ---------- Stage 2: PHP 8.4 + OpenSwoole + ext-zealphp runtime ----------
FROM php:8.4-cli-bookworm AS runtime-base
ENV OPENSWOOLE_VERSION=22.1.2
RUN apt-get update && apt-get install -y --no-install-recommends \
      git unzip libssl-dev libcurl4-openssl-dev libpcre3-dev \
      libbrotli-dev libnghttp2-dev ca-certificates \
    && docker-php-ext-install sockets \
    # OpenSwoole 22.1+ with HTTP/2, OpenSSL, cURL hooks
    && pecl install openswoole-${OPENSWOOLE_VERSION} \
    && docker-php-ext-enable openswoole \
    # MongoDB + Redis drivers
    && pecl install mongodb redis \
    && docker-php-ext-enable mongodb redis \
    # ext-zealphp: per-coroutine superglobal & session isolation
    && git clone --depth 1 https://github.com/sibidharan/zealphp /usr/src/zealphp \
    && cd /usr/src/zealphp/ext && phpize && ./configure && make -j"$(nproc)" && make install \
    && docker-php-ext-enable zealphp \
    && apt-get purge -y --auto-remove git \
    && rm -rf /var/lib/apt/lists/* /usr/src/zealphp

# Production PHP ini: opcache on, JIT tuned for a long-running server
COPY docker/php/production.ini /usr/local/etc/php/conf.d/zz-goco.ini

# ---------- Stage 3: Application image ----------
FROM runtime-base AS app
WORKDIR /app
# Non-root runtime user
RUN groupadd -g 10001 goco && useradd -u 10001 -g goco -s /usr/sbin/nologin goco
COPY --chown=goco:goco --from=vendor /build/vendor ./vendor
COPY --chown=goco:goco . .
# Writable runtime dirs (logs land in /tmp/zealphp by ZealPHP convention)
RUN mkdir -p storage /tmp/zealphp && chown -R goco:goco storage /tmp/zealphp
USER goco
EXPOSE 8080
ENV GOCO_ENV=production \
    ZEALPHP_MODE=coroutine
# app.php calls App::init('0.0.0.0', 8080)->run()
ENTRYPOINT ["php", "app.php"]
```

> **Note**
> The `ENTRYPOINT` runs `app.php` in the **foreground** on purpose. The ZealPHP process CLI (`start -d`, `restart`, `stop`, `status`, `logs`) is for bare-metal/manual operation; inside a container the process must be PID-managed by Docker so that `SIGTERM` triggers OpenSwoole's graceful drain and Docker's restart policy applies.

### `Dockerfile.dev` (dev tools + hot reload)

The dev image extends `runtime-base` but keeps dev Composer packages, adds Xdebug, and runs the app under a file watcher that restarts OpenSwoole workers on source change.

```dockerfile
# syntax=docker/dockerfile:1.7
FROM runtime-base AS dev
WORKDIR /app

# Dev tooling: Xdebug, Composer, watcher
RUN pecl install xdebug && docker-php-ext-enable xdebug
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
COPY docker/php/dev.ini /usr/local/etc/php/conf.d/zz-goco-dev.ini

# Dev deps installed at build; source is bind-mounted at runtime
COPY composer.json composer.lock ./
RUN composer install --no-scripts --no-interaction --prefer-dist

ENV GOCO_ENV=development \
    ZEALPHP_MODE=coroutine \
    XDEBUG_MODE=develop,debug \
    GOCO_HOT_RELOAD=1
EXPOSE 8080

# `goco serve --watch` restarts OpenSwoole workers on *.php change
ENTRYPOINT ["php", "app.php"]
CMD ["--watch"]
```

> **Tip**
> Hot reload works because the dev Compose overlay bind-mounts your working tree over `/app`. OpenSwoole's `reload_async` is triggered by the watcher, so in-flight requests finish on the old workers while new workers pick up the changed code — no dropped connections during development.

### `.dockerignore`

Keeps build context small and reproducible, and prevents host artifacts from leaking into the image.

```gitignore
# Version control & CI
.git
.github
.gitignore

# Local dependency & build output (installed inside the image)
vendor
node_modules
storage/logs/*
storage/cache/*
storage/tmp/*

# Env & secrets — provided at runtime, never baked in
.env
.env.*
!.env.example

# Docker & compose (not needed inside the image)
docker-compose*.yml
Dockerfile*
.dockerignore

# Editor / OS noise
.idea
.vscode
*.log
.DS_Store

# Tests & docs (excluded from the production image)
tests
docs
examples
```

> **Warning**
> `.env` is deliberately ignored. Never bake secrets into an image layer — layers are cacheable and shippable. Inject configuration at runtime via Compose `environment:` / `env_file:` or a secrets manager, as documented in [../getting-started/configuration.md](../getting-started/configuration.md).

---

## Compose Files

GOCO uses a three-file overlay strategy. The base is environment-neutral; overlays are additive.

| File | Purpose | How it is used |
|------|---------|----------------|
| `docker-compose.yml` | Base stack: all services, networks, volumes, healthchecks, restart policies. | Always loaded first. |
| `docker-compose.dev.yml` | Developer overlay: builds `Dockerfile.dev`, bind-mounts source, exposes DB/UI ports, enables Mailpit + hot reload. | `-f docker-compose.yml -f docker-compose.dev.yml` |
| `docker-compose.prod.yml` | Production overlay: pins the published image tag, hardens Traefik (Let's Encrypt, HTTP/3), enables Watchtower, sets replica/resource limits. | `-f docker-compose.yml -f docker-compose.prod.yml` |

```mermaid
flowchart LR
    base[docker-compose.yml<br/>base truth] --> dev[+ docker-compose.dev.yml<br/>build dev image, bind mounts, open ports]
    base --> prod[+ docker-compose.prod.yml<br/>pinned image, TLS, Watchtower, limits]
```

An `.env` file next to the Compose files supplies interpolated values (`${MONGO_ROOT_PASSWORD}`, `${GOCO_IMAGE_TAG}`, `${ACME_EMAIL}`, …). Compose reads it automatically.

---

## Services

Every service below declares four things GOCO treats as mandatory: a **healthcheck**, a **restart policy**, its **key environment**, and a **graceful-shutdown** contract (`stop_grace_period` plus a signal the process honors).

### Service matrix

| Compose service | Image | Purpose | Restart policy | Healthcheck | Stop signal / grace |
|-----------------|-------|---------|----------------|-------------|---------------------|
| `gococms` | built from `Dockerfile` | ZealPHP/OpenSwoole application server | `unless-stopped` | HTTP `GET /healthz` | `SIGTERM` / 30s drain |
| `mongodb` | `mongo:7` | Primary datastore (documents, transactions, indexes, text search) | `unless-stopped` | `mongosh ping` | `SIGTERM` / 30s |
| `redis` | `redis:7-alpine` | Cache, queue, sessions, locks, rate-limit, pub/sub | `unless-stopped` | `redis-cli PING` | `SIGTERM` / 15s |
| `traefik` | `traefik:v3` | Reverse proxy, TLS (Let's Encrypt), HTTP/3, dynamic routing | `unless-stopped` | `traefik healthcheck --ping` | `SIGTERM` / 15s |
| `minio` | `minio/minio` | S3-compatible object storage (media, backups) | `unless-stopped` | `mc ready` / `/minio/health/live` | `SIGTERM` / 20s |
| `meilisearch` | `getmeili/meilisearch:v1` | Search provider (swappable) | `unless-stopped` | `GET /health` | `SIGTERM` / 15s |
| `mailpit` | `axllent/mailpit` | Dev mail capture (SMTP + web UI) | `unless-stopped` (dev only) | `GET /readyz` | `SIGTERM` / 5s |
| `watchtower` | `containrrr/watchtower` | **Optional** automatic image updates | `unless-stopped` | process check | `SIGTERM` / 10s |

### `gococms` — the application

The one moving part that scales horizontally. It talks to every backend over the private `goco_internal` network and is the only service Traefik routes public traffic to. It exposes a lightweight `/healthz` route (a ZealPHP `$app->route('/healthz', ...)` handler that pings Redis + MongoDB and returns `200`).

```yaml
  gococms:
    build:
      context: .
      dockerfile: Dockerfile
      target: app
    image: ${GOCO_IMAGE:-gococms/core}:${GOCO_IMAGE_TAG:-latest}
    restart: unless-stopped
    depends_on:
      mongodb:   { condition: service_healthy }
      redis:     { condition: service_healthy }
      minio:     { condition: service_healthy }
    environment:
      GOCO_ENV: production
      ZEALPHP_MODE: coroutine            # App::MODE_COROUTINE
      APP_HOST: 0.0.0.0
      APP_PORT: "8080"
      MONGODB_URI: mongodb://gococms:${MONGO_APP_PASSWORD}@mongodb:27017/goco?authSource=goco
      REDIS_URL: redis://redis:6379/0
      STORAGE_DRIVER: minio              # local | minio | s3
      MINIO_ENDPOINT: http://minio:9000
      MINIO_ACCESS_KEY: ${MINIO_ROOT_USER}
      MINIO_SECRET_KEY: ${MINIO_ROOT_PASSWORD}
      SEARCH_DRIVER: meilisearch         # mongodb | meilisearch | opensearch
      MEILI_HOST: http://meilisearch:7700
      MEILI_MASTER_KEY: ${MEILI_MASTER_KEY}
    healthcheck:
      test: ["CMD", "php", "-r", "exit(@file_get_contents('http://127.0.0.1:8080/healthz') ? 0 : 1);"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 20s
    stop_grace_period: 30s               # OpenSwoole drains in-flight coroutines on SIGTERM
    networks: [goco_internal, goco_edge]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.gococms.rule=Host(`${PRIMARY_DOMAIN}`)"
      - "traefik.http.routers.gococms.entrypoints=websecure"
      - "traefik.http.routers.gococms.tls.certresolver=letsencrypt"
      - "traefik.http.services.gococms.loadbalancer.server.port=8080"
```

> **Note**
> `stop_grace_period: 30s` is the graceful-shutdown contract. On `docker stop`, Docker sends `SIGTERM`; OpenSwoole stops accepting new connections, lets running coroutines finish, flushes logs to `/tmp/zealphp/`, and exits before the 30s SIGKILL deadline. Keep this longer than your slowest expected request.

Detailed Traefik labels (per-tenant routers, wildcard certs, HTTP/3, security-header middleware) live in [traefik.md](traefik.md).

### `mongodb` — primary datastore

Holds every collection in the GOCO data model (`workspaces`, `websites`, `pages`, `posts`, `media`, `audit_logs`, …). Uses JSON-Schema validators, aggregation pipelines, and multi-document transactions, so it must run as a **replica set** (even single-node) to enable transactions in production.

```yaml
  mongodb:
    image: mongo:7
    restart: unless-stopped
    command: ["--replSet", "goco-rs", "--bind_ip_all", "--wiredTigerCacheSizeGB", "1"]
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: goco
    volumes:
      - mongo_data:/data/db
      - mongo_config:/data/configdb
      - ./docker/mongo/init:/docker-entrypoint-initdb.d:ro   # replset init + app user + indexes
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s
    stop_grace_period: 30s
    networks: [goco_internal]
```

> **Warning**
> Multi-document transactions — used by GOCO to keep cross-collection invariants (e.g. publishing a page while writing a `page_revisions` doc and an `audit_logs` entry) — require a replica set. A standalone `mongod` will reject `startSession()` transactions. The `--replSet goco-rs` flag plus a one-time `rs.initiate()` in the init script satisfies this.

### `redis` — cache, queue, realtime, locks

Backs the cache layer, the job queue, per-coroutine sessions, distributed locks, rate limiting, and pub/sub for realtime. AOF persistence is enabled so queued jobs survive a restart. See [../architecture/caching-and-queue.md](../architecture/caching-and-queue.md).

```yaml
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes", "--maxmemory-policy", "noeviction", "--requirepass", "${REDIS_PASSWORD}"]
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "PING"]
      interval: 10s
      timeout: 3s
      retries: 5
    stop_grace_period: 15s
    networks: [goco_internal]
```

> **Note**
> `maxmemory-policy noeviction` protects the queue: GOCO uses Redis as durable job storage, so silently evicting keys under memory pressure would drop jobs. If you run a **separate** Redis purely for cache, set that one to `allkeys-lru`.

### `traefik` — reverse proxy

The only public listener. Terminates TLS via Let's Encrypt, serves HTTP/3 (QUIC on UDP 443), discovers routers from Docker labels, and applies middleware (rate limits, security headers, redirects). Full configuration — entrypoints, cert resolvers, wildcard/multi-domain, per-tenant routers — is documented in [traefik.md](traefik.md).

```yaml
  traefik:
    image: traefik:v3
    restart: unless-stopped
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.websecure.http3=true"
      - "--certificatesresolvers.letsencrypt.acme.email=${ACME_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.storage=/acme/acme.json"
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"          # HTTP/3
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_acme:/acme
    healthcheck:
      test: ["CMD", "traefik", "healthcheck", "--ping"]
      interval: 15s
      timeout: 5s
      retries: 5
    stop_grace_period: 15s
    networks: [goco_edge]
```

### `minio` — object storage

S3-compatible storage behind GOCO's storage driver interface (`local | minio | s3`). Stores uploaded media, generated derivatives, and backup artifacts. Buckets are provisioned by an init sidecar (`minio/mc`).

```yaml
  minio:
    image: minio/minio:latest
    restart: unless-stopped
    command: ["server", "/data", "--console-address", ":9001"]
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 20s
    stop_grace_period: 20s
    networks: [goco_internal]
```

> **Tip**
> In production you often skip this service entirely and point `STORAGE_DRIVER=s3` at Amazon S3. The driver interface means the app code is identical — only environment changes. MinIO gives you an S3 target on your own hardware and for local/offline development.

### `meilisearch` — search

Default search provider (the interface also supports MongoDB text/Atlas Search and OpenSearch). Indexes pages, posts, and collection entries for the site search and admin quick-find.

```yaml
  meilisearch:
    image: getmeili/meilisearch:v1
    restart: unless-stopped
    environment:
      MEILI_MASTER_KEY: ${MEILI_MASTER_KEY}
      MEILI_ENV: production
      MEILI_NO_ANALYTICS: "true"
    volumes:
      - meili_data:/meili_data
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://127.0.0.1:7700/health"]
      interval: 15s
      timeout: 5s
      retries: 5
    stop_grace_period: 15s
    networks: [goco_internal]
```

### `mailpit` — dev mail

Captures all outbound mail in development so transactional emails (password resets, invites, form notifications) never reach real inboxes. Provides an SMTP endpoint on `1025` and a web UI on `8025`. Enabled by the **dev** overlay only.

```yaml
  mailpit:
    image: axllent/mailpit:latest
    restart: unless-stopped
    environment:
      MP_SMTP_AUTH_ACCEPT_ANY: "1"
      MP_SMTP_AUTH_ALLOW_INSECURE: "1"
    ports:
      - "8025:8025"     # web UI
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://127.0.0.1:8025/readyz"]
      interval: 15s
      timeout: 3s
      retries: 5
    stop_grace_period: 5s
    networks: [goco_internal]
```

In development the app sends via `MAIL_DSN=smtp://mailpit:1025`. Production points `MAIL_DSN` at your real provider.

### `watchtower` — optional auto-update

Watches running containers and pulls new images when a tag is updated, then recreates the container with the same config. Enabled by the **prod** overlay and strictly opt-in.

```yaml
  watchtower:
    image: containrrr/watchtower:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      WATCHTOWER_CLEANUP: "true"
      WATCHTOWER_LABEL_ENABLE: "true"      # only update containers opting in
      WATCHTOWER_POLL_INTERVAL: "300"
    command: ["--label-enable"]
    stop_grace_period: 10s
    networks: [goco_internal]
```

> **Warning**
> Auto-update is convenient but can roll a breaking image into production unattended. For anything beyond a hobby deployment, pin `GOCO_IMAGE_TAG` to an immutable version and promote deliberately (see [deployment-guide.md](deployment-guide.md)) instead of tracking `latest` with Watchtower.

---

## Networks

Two networks enforce a security boundary: only Traefik and the app straddle the edge; the datastores are unreachable from outside.

```yaml
networks:
  goco_edge:
    driver: bridge          # traefik <-> gococms (public-facing tier)
  goco_internal:
    driver: bridge
    internal: true          # no route to the outside world
```

| Network | Members | Reachable from host/internet |
|---------|---------|------------------------------|
| `goco_edge` | `traefik`, `gococms` | Only via Traefik's published `80/443` ports |
| `goco_internal` | `gococms`, `mongodb`, `redis`, `minio`, `meilisearch`, `mailpit`, `watchtower` | No (`internal: true`) |

> **Note**
> Marking `goco_internal` as `internal: true` means MongoDB, Redis, and MinIO have **no** route to the internet and are not exposed on the host at all in production. The app reaches them by Docker DNS service name (`mongodb`, `redis`, …). Ports for these services are published only by the **dev** overlay, for local inspection.

---

## Volumes

All persistent state lives in named volumes so the app container stays disposable.

```yaml
volumes:
  mongo_data:      # /data/db      — collections, indexes
  mongo_config:    # /data/configdb — replica-set metadata
  redis_data:      # /data         — AOF (queue durability)
  minio_data:      # /data         — objects / media
  meili_data:      # /meili_data   — search indexes
  traefik_acme:    # /acme         — Let's Encrypt certificates
```

| Volume | Service | Contains | Backed up? |
|--------|---------|----------|-----------|
| `mongo_data` | mongodb | All documents & indexes | **Yes** (primary) |
| `mongo_config` | mongodb | Replica-set config | Yes |
| `redis_data` | redis | AOF — pending jobs | Yes (jobs) |
| `minio_data` | minio | Media objects | **Yes** (unless using external S3) |
| `meili_data` | meilisearch | Search indexes | Rebuildable from Mongo |
| `traefik_acme` | traefik | TLS certs (`acme.json`) | Yes (avoids re-issuing) |

Backup and restore procedures for these volumes are covered in [backup-restore.md](backup-restore.md).

---

## Build & Run Commands

### Development

```bash
# Build the dev image and start the full stack with hot reload
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build

# Tail the app logs (ZealPHP also writes to /tmp/zealphp/ inside the container)
docker compose logs -f gococms

# Run the GOCO CLI inside the app container (migrations, generators, lifecycle)
docker compose exec gococms php goco migrate
docker compose exec gococms php goco make:widget HeroBanner

# One-time MongoDB replica-set init (idempotent; usually run by the init script)
docker compose exec mongodb mongosh --eval "rs.initiate({_id:'goco-rs',members:[{_id:0,host:'mongodb:27017'}]})"

# Open the dev mail UI: http://localhost:8025  (Mailpit)
```

### Production

```bash
# Pull the pinned, published image and start with the prod overlay
export GOCO_IMAGE_TAG=1.0.0-rc.3
docker compose -f docker-compose.yml -f docker-compose.prod.yml pull
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Verify health of every service
docker compose ps
docker compose -f docker-compose.yml -f docker-compose.prod.yml exec gococms php goco doctor

# Scale the app tier horizontally (Traefik load-balances automatically)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --scale gococms=3

# Graceful rolling restart (SIGTERM drains OpenSwoole coroutines)
docker compose -f docker-compose.yml -f docker-compose.prod.yml restart gococms
```

> **Tip**
> Wrap the two `-f` flags in a shell alias or a `Makefile` target (`make up`, `make prod`) so nobody forgets the overlay. A missing overlay is the most common cause of "it started but has no TLS / no image tag."

### Building & publishing the production image

```bash
# Build only the production target and tag it
docker build --target app -t gococms/core:1.0.0-rc.3 .

# Multi-arch build (amd64 + arm64) for a mixed fleet
docker buildx build --platform linux/amd64,linux/arm64 \
  --target app -t gococms/core:1.0.0-rc.3 --push .
```

---

## Environment & Configuration

Every `${VAR}` in the Compose files resolves from the `.env` file beside them. The canonical list — names, defaults, and meaning — is maintained in [../getting-started/configuration.md](../getting-started/configuration.md) and [../reference/configuration-reference.md](../reference/configuration-reference.md). A minimal production `.env`:

```env
# Image
GOCO_IMAGE=gococms/core
GOCO_IMAGE_TAG=1.0.0-rc.3

# Edge
PRIMARY_DOMAIN=example.com
ACME_EMAIL=ops@example.com

# MongoDB
MONGO_ROOT_USER=root
MONGO_ROOT_PASSWORD=change-me-strong
MONGO_APP_PASSWORD=change-me-strong-too

# Redis
REDIS_PASSWORD=change-me-redis

# MinIO / object storage
MINIO_ROOT_USER=goco
MINIO_ROOT_PASSWORD=change-me-minio

# Search
MEILI_MASTER_KEY=change-me-meili
```

> **Warning**
> The values above are placeholders. Generate strong, unique secrets per environment (`openssl rand -hex 32`) and keep `.env` out of version control — it is already listed in both `.gitignore` and `.dockerignore`.

---

## Operational Notes

- **Logs.** OpenSwoole logs to stdout (captured by `docker compose logs`) and additionally to `/tmp/zealphp/` inside the container per ZealPHP convention. Ship stdout to your log aggregator; treat `/tmp/zealphp/` as ephemeral.
- **Health gating.** `depends_on: { condition: service_healthy }` ensures the app never boots before MongoDB, Redis, and MinIO report healthy — avoiding a crash loop on cold start.
- **Resource limits.** The prod overlay sets `deploy.resources.limits` per service; tune MongoDB's `--wiredTigerCacheSizeGB` and OpenSwoole worker counts to the host. See [scaling.md](scaling.md).
- **Zero-downtime deploys.** Because the app drains on `SIGTERM` and Traefik health-checks backends, a `--scale`-based rolling replace serves traffic uninterrupted. Full procedure in [deployment-guide.md](deployment-guide.md).

---

## Related

- [Traefik Reverse Proxy](traefik.md)
- [Deployment Guide](deployment-guide.md)
- [Backup & Restore](backup-restore.md)
- [Scaling Strategy](scaling.md)
- [Configuration](../getting-started/configuration.md)
- [Configuration Reference](../reference/configuration-reference.md)
- [Caching, Queue & Realtime (Redis)](../architecture/caching-and-queue.md)
- [Storage & Media](../architecture/storage.md)
- [Search](../architecture/search.md)
- [Documentation Home](../README.md)
