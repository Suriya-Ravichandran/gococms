# Deployment Guide

> Take GOCO CMS from a clean host to a production, TLS-terminated, zero-downtime deployment — with migrations, healthchecks, observability, upgrades, and rollback.

This guide is the end-to-end runbook for deploying GOCO CMS in production. It assumes a Docker-first
topology: the `gococms` application container (ZealPHP on OpenSwoole) behind [Traefik](./traefik.md), with
[MongoDB](../architecture/database-mongodb.md) and Redis as stateful backing services. Non-Docker (bare-metal)
and Kubernetes notes are included at the end.

`stable` — applies to GOCO CMS pre-1.0 releases. Pin an exact image tag in production; never track `latest`.

---

## Table of Contents

1. [Deployment Topology](#deployment-topology)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Prepare the Host](#step-1--prepare-the-host)
4. [Step 2 — Configure DNS](#step-2--configure-dns)
5. [Step 3 — Prepare the Environment File](#step-3--prepare-the-environment-file)
6. [Step 4 — Pull or Build Images](#step-4--pull-or-build-images)
7. [Step 5 — Bring the Stack Up](#step-5--bring-the-stack-up)
8. [Step 6 — Migrate and Seed](#step-6--migrate-and-seed)
9. [Step 7 — First-Run Installer](#step-7--first-run-installer)
10. [Step 8 — Verify Health and TLS](#step-8--verify-health-and-tls)
11. [Zero-Downtime Deploys](#zero-downtime-deploys)
12. [Observability](#observability)
13. [Secrets Management](#secrets-management)
14. [Upgrade Procedure](#upgrade-procedure)
15. [Rollback](#rollback)
16. [Bare-Metal Deployment](#bare-metal-deployment)
17. [Kubernetes Deployment](#kubernetes-deployment)
18. [Production Checklist](#production-checklist)
19. [Related](#related)

---

## Deployment Topology

```mermaid
flowchart TB
    Internet(("Internet<br/>HTTP/3 · TLS")) --> TR[Traefik<br/>:80 · :443]
    TR -->|drain-aware<br/>load balance| A1[gococms replica 1]
    TR --> A2[gococms replica 2]
    subgraph app[Stateless app tier]
      A1
      A2
    end
    A1 --> MON[(MongoDB<br/>replica set)]
    A2 --> MON
    A1 --> RD[(Redis<br/>cache · queue · sessions)]
    A2 --> RD
    A1 --> OBJ[(Object storage<br/>MinIO / S3)]
    A2 --> OBJ
    A1 --> SR[(Search<br/>Meilisearch)]
    A2 --> SR
    TR -.ACME / Let's Encrypt.-> ACME[[acme.json]]
```

The application tier is **stateless**: every replica reads and writes the same MongoDB, Redis, object store, and
search cluster. Sessions live in Redis, not in worker memory, so any replica can serve any request. This is the
property that makes rolling deploys and horizontal scaling safe — see [Scaling Strategy](./scaling.md).

---

## Prerequisites

| Requirement | Minimum | Recommended (production) |
| --- | --- | --- |
| Host OS | Linux kernel 5.10+ (Ubuntu 22.04 LTS / Debian 12) | Ubuntu 24.04 LTS |
| CPU | 2 vCPU | 4+ vCPU (OpenSwoole scales with cores) |
| RAM | 4 GB | 8 GB+ (MongoDB working set + Redis) |
| Disk | 20 GB SSD | 100 GB+ NVMe, separate volume for MongoDB |
| Docker Engine | 24.0+ | Latest stable, with Compose v2 plugin |
| Docker Compose | v2.20+ (`docker compose`) | Latest |
| Public IPv4 (and IPv6) | 1 static address | Static A/AAAA records |
| Ports open | 80/tcp, 443/tcp, 443/udp (HTTP/3) | Same; SSH restricted to admin CIDR |
| PHP (bare-metal only) | 8.4+ with OpenSwoole 22.1+ | 8.4+ |

> **Note**
> Ports **80** and **443/tcp** are required for Let's Encrypt HTTP-01/TLS challenges and normal traffic.
> Port **443/udp** must be open for HTTP/3 (QUIC). MongoDB (27017) and Redis (6379) must **never** be exposed
> publicly — keep them on the internal Docker network or a private subnet.

Confirm the toolchain:

```bash
docker --version          # Docker Engine 24.0+
docker compose version    # Compose v2.20+
uname -r                  # Kernel 5.10+
```

---

## Step 1 — Prepare the Host

Install Docker Engine and the Compose plugin, create a non-root deploy user, and raise the kernel limits
OpenSwoole needs for high connection counts.

```bash
# Install Docker Engine + Compose plugin (Debian/Ubuntu)
curl -fsSL https://get.docker.com | sh

# Run the app as a dedicated, non-root user
sudo useradd -m -s /bin/bash -G docker deploy
sudo -iu deploy

# Raise file-descriptor limits for OpenSwoole (many concurrent coroutines/connections)
echo 'fs.file-max = 1048576' | sudo tee /etc/sysctl.d/60-gococms.conf
echo 'net.core.somaxconn = 65535' | sudo tee -a /etc/sysctl.d/60-gococms.conf
sudo sysctl --system
```

Clone the deployment repository (or your fork of the compose bundle) into the deploy user's home:

```bash
git clone https://github.com/gococms/gococms.git /home/deploy/gococms
cd /home/deploy/gococms
git checkout v0.9.0    # pin an exact release tag — never deploy from a moving branch
```

The Docker services, healthchecks, and Traefik labels referenced below are defined in the compose files; the full
service reference lives in [Docker Architecture](./docker.md).

---

## Step 2 — Configure DNS

GOCO CMS is multi-domain by design (one deployment serves many websites across workspaces). Point every hostname
the platform will terminate at the host's public IP.

| Record | Type | Value | Purpose |
| --- | --- | --- | --- |
| `cms.example.com` | A / AAAA | host IP | Admin + API control plane |
| `www.example.com` | A / AAAA | host IP | A tenant website |
| `*.example.com` | A / AAAA | host IP | Wildcard for tenant subdomains |
| `traefik.example.com` | A / AAAA | host IP | Traefik dashboard (protect it) |

> **Tip**
> For per-tenant custom domains, either add individual A/AAAA records or use a wildcard plus a
> [Traefik](./traefik.md) `HostRegexp` router. Wildcard **certificates** require the DNS-01 challenge and a
> supported DNS provider; single hostnames can use the simpler HTTP-01 challenge.

Verify propagation before continuing (ACME issuance will fail otherwise):

```bash
dig +short A cms.example.com
dig +short AAAA cms.example.com
```

---

## Step 3 — Prepare the Environment File

All configuration is injected via environment variables (see the full
[Configuration Reference](../reference/configuration-reference.md)). Start from the template and fill in real
secrets — never commit the resulting `.env`.

```bash
cp .env.example .env
chmod 600 .env
```

Generate strong secrets:

```bash
# APP_KEY, JWT secret, and per-service passwords
openssl rand -base64 48   # run once per secret
```

A production `.env`:

```env
# ---- Application ----
APP_ENV=production
APP_DEBUG=false
APP_URL=https://cms.example.com
APP_KEY=base64:REPLACE_WITH_openssl_rand_base64_48
ZEALPHP_MODE=coroutine            # MODE_COROUTINE — the modern default
ZEALPHP_HOST=0.0.0.0
ZEALPHP_PORT=8080
ZEALPHP_WORKERS=0                 # 0 = auto (one worker per CPU core)

# ---- Domains / TLS ----
PRIMARY_DOMAIN=cms.example.com
ACME_EMAIL=ops@example.com        # Let's Encrypt registration + expiry alerts
ACME_CA=production                # 'staging' while testing to avoid rate limits

# ---- MongoDB ----
MONGODB_URI=mongodb://gococms:REPLACE_STRONG_PW@mongodb:27017/goco?replicaSet=rs0&authSource=admin
MONGODB_DB=goco

# ---- Redis (cache / queue / sessions / locks / rate-limit / realtime) ----
REDIS_URL=redis://:REPLACE_STRONG_PW@redis:6379/0

# ---- Auth ----
JWT_SECRET=REPLACE_WITH_openssl_rand_base64_48
SESSION_DRIVER=redis
PASSWORD_HASH=argon2id

# ---- Object storage (local | minio | s3) ----
STORAGE_DRIVER=minio
STORAGE_ENDPOINT=http://minio:9000
STORAGE_BUCKET=gococms-media
STORAGE_KEY=REPLACE_ACCESS_KEY
STORAGE_SECRET=REPLACE_SECRET_KEY

# ---- Search (mongodb | meilisearch | opensearch) ----
SEARCH_DRIVER=meilisearch
SEARCH_HOST=http://meilisearch:7700
SEARCH_KEY=REPLACE_MASTER_KEY

# ---- Mail ----
MAIL_DRIVER=smtp
MAIL_HOST=mailpit          # dev/staging; use a real SMTP relay in production
MAIL_PORT=1025
```

> **Warning**
> Keep `ACME_CA=staging` during your first bring-up. Let's Encrypt's production endpoint enforces strict
> per-domain rate limits, and a misconfigured DNS record can burn your weekly quota. Switch to `production`
> only once staging certs issue cleanly, then delete `acme.json` so real certs are re-requested.

> **Note**
> `MONGODB_URI` must point at a **replica set** (even a single-node one, `rs0`) — multi-document transactions,
> which GOCO uses to keep cross-collection invariants consistent, require a replica set. See the
> [MongoDB Data Layer](../architecture/database-mongodb.md).

---

## Step 4 — Pull or Build Images

Prefer pulling the published, tagged image. Build locally only when you maintain a fork.

```bash
# Pull pinned release images defined in the production compose file
docker compose -f docker-compose.prod.yml pull

# --- or --- build from source (fork/customization)
docker compose -f docker-compose.prod.yml build --pull
```

Confirm the resolved image digests so you know exactly what will run:

```bash
docker compose -f docker-compose.prod.yml config | grep -E 'image:'
```

---

## Step 5 — Bring the Stack Up

Start the backing services and the app tier in detached mode. Traefik claims 80/443 and begins the ACME flow.

```bash
docker compose -f docker-compose.prod.yml up -d
```

Watch the services converge to healthy:

```bash
docker compose -f docker-compose.prod.yml ps
```

Expected services (compose names): `gococms`, `mongodb`, `redis`, `traefik`, `minio`, `meilisearch`,
`mailpit`, and optionally `watchtower`. Each ships a healthcheck and a `restart: unless-stopped` policy — the
details are in [Docker Architecture](./docker.md).

> **Note**
> On first boot MongoDB may need its replica set initiated. If your compose file does not auto-initiate:
> ```bash
> docker compose -f docker-compose.prod.yml exec mongodb \
>   mongosh --eval 'rs.initiate({_id:"rs0",members:[{_id:0,host:"mongodb:27017"}]})'
> ```

---

## Step 6 — Migrate and Seed

Migrations create collections, apply JSON-Schema validators, and build the documented indexes; seeding installs
the default roles, capabilities, and system settings. Run both through the `goco` CLI inside the app container.

```bash
# Apply schema validators + indexes (idempotent, safe to re-run)
docker compose -f docker-compose.prod.yml exec gococms goco migrate

# Seed roles, capabilities, default settings
docker compose -f docker-compose.prod.yml exec gococms goco db:seed
```

`goco migrate` is idempotent and forward-only: it records applied migrations in a `migrations` bookkeeping
document and skips anything already applied. Re-running it on every deploy is expected and safe. The full command
surface is in the [CLI Reference](../reference/cli-reference.md).

> **Tip**
> In an automated pipeline, run `goco migrate` as a one-shot task **before** rolling new app replicas so the new
> code always meets a schema it understands. GOCO migrations are additive by convention (new fields/indexes,
> never destructive drops in the same release), which keeps old and new replicas compatible during a rollout.

---

## Step 7 — First-Run Installer

The `installer` app finalizes the deployment: it creates the first **Workspace**, the owner account, and the
initial **Website**. Complete it via the web UI or headlessly.

Browser flow — visit `https://cms.example.com/install` and provide:

- Owner email, display name, and a strong password (hashed with Argon2id).
- Workspace name and slug.
- First website name and its primary domain.

Headless flow (CI / immutable infra):

```bash
docker compose -f docker-compose.prod.yml exec gococms goco install \
  --owner-email="owner@example.com" \
  --owner-password-env=OWNER_BOOTSTRAP_PW \
  --workspace="Acme" \
  --website="Acme Marketing" \
  --domain="www.example.com"
```

Once the owner exists, the `/install` route is disabled automatically. See the
[Authentication](../core/authentication.md) doc for enabling 2FA (TOTP) and Passkeys (WebAuthn) on the owner
account immediately after install.

---

## Step 8 — Verify Health and TLS

**Container health** — all services should report `healthy`:

```bash
docker compose -f docker-compose.prod.yml ps
docker inspect --format '{{.State.Health.Status}}' $(docker compose -f docker-compose.prod.yml ps -q gococms)
```

**Application health endpoints** — the app exposes a liveness and a readiness probe:

```bash
# Liveness: process is up
curl -fsS https://cms.example.com/healthz

# Readiness: MongoDB, Redis, storage, search all reachable
curl -fsS https://cms.example.com/readyz | jq
```

A healthy `/readyz` returns each dependency's status:

```json
{
  "status": "ok",
  "checks": {
    "mongodb": "ok",
    "redis": "ok",
    "storage": "ok",
    "search": "ok"
  },
  "version": "0.9.0"
}
```

**TLS / Traefik** — confirm a real (non-staging) certificate is served over HTTP/2 and HTTP/3:

```bash
# Certificate issuer + validity
echo | openssl s_client -connect cms.example.com:443 -servername cms.example.com 2>/dev/null \
  | openssl x509 -noout -issuer -dates

# HTTP/3 advertised via alt-svc
curl -sI https://cms.example.com | grep -i alt-svc
```

If the certificate shows Let's Encrypt as the issuer and `/readyz` is `ok`, the deployment is live. Traefik
routing, middleware, and security headers are covered in [Traefik Reverse Proxy](./traefik.md).

---

## Zero-Downtime Deploys

GOCO's stateless app tier plus OpenSwoole's graceful reload make rolling deploys safe. Two complementary
mechanisms:

**1. OpenSwoole graceful reload (in place).** Sending `SIGUSR1`/`SIGUSR2` to the master tells OpenSwoole to
reload worker processes: existing coroutines finish, in-flight requests drain, then workers restart with new
code. The `goco` process wrapper exposes this as `reload`:

```bash
docker compose -f docker-compose.prod.yml exec gococms goco reload
```

Use graceful reload for **code changes that ship in the same image build** (rare in Docker) or after config
changes that a worker picks up on restart.

**2. Rolling replica replacement (image swap).** For a new image tag, replace replicas one at a time so Traefik
always has healthy backends. The order matters: migrate first (additive), then roll.

```bash
# 1) Run additive migrations against the shared DB (old replicas keep serving)
docker compose -f docker-compose.prod.yml run --rm gococms goco migrate

# 2) Pull the new pinned image
docker compose -f docker-compose.prod.yml pull gococms

# 3) Scale up with the new image, let healthchecks pass, then scale old out
docker compose -f docker-compose.prod.yml up -d --no-deps --scale gococms=4 gococms
# Compose recreates gococms replicas; with a healthcheck + start_period Traefik
# only routes to replicas reporting healthy, and drains draining ones.
```

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant DB as MongoDB
    participant TR as Traefik
    participant Old as Replica (old)
    participant New as Replica (new)
    CI->>DB: goco migrate (additive)
    CI->>New: start new replica (image vNext)
    New->>TR: healthcheck passes -> added to pool
    CI->>Old: SIGTERM (graceful shutdown)
    Old->>TR: stop accepting; drain in-flight coroutines
    Old-->>TR: removed from pool
    Note over TR: Zero dropped requests — pool never empty
```

Key requirements for zero downtime:

- **Graceful shutdown window.** Give the container time to drain. Set a generous stop grace period so OpenSwoole
  can finish in-flight requests before `SIGKILL`:
  ```yaml
  services:
    gococms:
      stop_grace_period: 30s   # OpenSwoole drains coroutines on SIGTERM within this window
  ```
- **Healthcheck gating.** Traefik only load-balances to replicas whose `/readyz` passes; a `start_period` keeps
  a booting replica out of rotation until MongoDB/Redis are reachable.
- **Sticky-free sessions.** Because sessions live in Redis, Traefik does not need session affinity — any replica
  can serve any user, so draining one replica never logs anyone out.
- **Additive migrations only.** Never drop a field/collection an in-flight old replica still reads. Split
  destructive changes across two releases (add-and-backfill in vN, remove in vN+1).

See [Scaling Strategy](./scaling.md) for replica sizing and autoscaling signals.

---

## Observability

**Logs.** OpenSwoole writes structured logs; the ZealPHP process manager mirrors them under `/tmp/zealphp/`
inside the container, and Docker captures stdout/stderr.

```bash
# Follow the app logs
docker compose -f docker-compose.prod.yml logs -f --tail=200 gococms

# Or via the process wrapper (reads /tmp/zealphp/*.log)
docker compose -f docker-compose.prod.yml exec gococms goco logs

# Traefik access + ACME logs
docker compose -f docker-compose.prod.yml logs -f traefik
```

Ship logs off-host to your aggregator (Loki, ELK, or a hosted service) with a Docker logging driver so restarts
don't lose history:

```yaml
services:
  gococms:
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "5" }
```

**Metrics.** Enable the Traefik Prometheus endpoint and scrape the app's own metrics route:

| Source | Endpoint | Signals |
| --- | --- | --- |
| Traefik | `:8082/metrics` (internal) | request rate, latency, 4xx/5xx, TLS cert expiry |
| GOCO app | `/metrics` (guarded, internal) | request duration, coroutine count, queue depth, cache hit rate |
| MongoDB | `mongodb_exporter` | ops/s, replication lag, connections, working-set |
| Redis | `redis_exporter` | memory, evictions, hit ratio, queue length |

**Healthchecks.** `/healthz` (liveness) and `/readyz` (readiness) are the same probes your orchestrator and
Traefik use. Alert on `/readyz` flipping to non-`ok`, on 5xx rate, on p99 latency, and on Let's Encrypt cert
expiry < 14 days.

> **Tip**
> Watch **Redis queue depth** and **MongoDB replication lag** as leading indicators. A rising queue means the
> worker pool can't keep up (scale app replicas); rising lag means MongoDB is I/O-bound (scale the DB tier).

---

## Secrets Management

Secrets never belong in the image or in version control. In order of preference:

1. **Docker/Swarm secrets or an orchestrator secret store.** Mount secrets as files and reference them; the app
   reads `*_FILE` variants when present:
   ```yaml
   services:
     gococms:
       environment:
         MONGODB_URI_FILE: /run/secrets/mongodb_uri
         JWT_SECRET_FILE: /run/secrets/jwt_secret
       secrets: [mongodb_uri, jwt_secret]
   secrets:
     mongodb_uri: { external: true }
     jwt_secret: { external: true }
   ```
2. **A dedicated secrets manager** (HashiCorp Vault, AWS/GCP Secrets Manager) injected at deploy time into the
   environment or into files under `/run/secrets`.
3. **A `.env` with `chmod 600`, owned by the deploy user**, outside the git working tree — the minimum bar for a
   single-host deployment.

Rules of the road:

- Rotate `APP_KEY`, `JWT_SECRET`, and database/Redis passwords on a schedule and after any staff change.
- Rotating `JWT_SECRET` invalidates outstanding API tokens by design — communicate the window, or support a
  two-key overlap during rotation.
- Never log secret values. `APP_DEBUG=false` in production keeps stack traces and env dumps out of responses.
- Restrict `.env`/secret file access to the deploy user; audit reads via `audit_logs`.

The complete list of configurable keys is the [Configuration Reference](../reference/configuration-reference.md).

---

## Upgrade Procedure

Upgrades follow **backup → pull → migrate → roll → verify**. Because migrations are additive and the app tier is
stateless, this is a rolling operation with no maintenance window for patch/minor releases.

```bash
# 0) Read the release notes / breaking changes first
#    https://github.com/gococms/gococms/releases  (and ../changelog.md)

# 1) BACK UP MongoDB + object storage BEFORE touching anything
#    (see ./backup-restore.md for the authoritative procedure)
docker compose -f docker-compose.prod.yml exec mongodb \
  mongodump --uri="$MONGODB_URI" --archive=/backup/pre-upgrade-$(date +%F).gz --gzip

# 2) Pin and pull the new release image
git -C /home/deploy/gococms fetch --tags
git -C /home/deploy/gococms checkout v0.10.0     # exact target tag
docker compose -f docker-compose.prod.yml pull

# 3) Apply migrations (additive; old replicas keep serving)
docker compose -f docker-compose.prod.yml run --rm gococms goco migrate

# 4) Roll app replicas onto the new image (see Zero-Downtime Deploys)
docker compose -f docker-compose.prod.yml up -d --no-deps gococms

# 5) Verify
curl -fsS https://cms.example.com/readyz | jq '.version, .status'
docker compose -f docker-compose.prod.yml ps
```

> **Warning**
> Always take a fresh backup **before** running migrations. GOCO migrations are forward-only; there is no
> automatic `down` migration. Recovery from a bad upgrade is a **restore from backup**, not a schema rollback —
> the full procedure is in [Backup & Restore](./backup-restore.md).

For upgrades that cross a documented breaking-change boundary, follow the two-phase pattern: deploy the
add-and-backfill release, let it run, then deploy the cleanup release that removes the deprecated fields.

---

## Rollback

Two independent axes — **application code** and **data** — roll back separately.

**Code rollback (fast, no data loss).** If a new image regresses but the schema is compatible (additive
migrations mean the old code still works against the new schema), redeploy the previous tag:

```bash
git -C /home/deploy/gococms checkout v0.9.0
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d --no-deps gococms
curl -fsS https://cms.example.com/readyz | jq '.version'
```

This is the common case and is why additive-only migrations are a hard rule: the previous release must remain
runnable against the migrated database.

**Data rollback (rare, destructive).** Only when a migration or bad write corrupted data. Stop the app tier,
restore MongoDB from the pre-upgrade backup, then bring back the matching code version:

```bash
# 1) Stop the app so nothing writes during restore
docker compose -f docker-compose.prod.yml stop gococms

# 2) Restore from the backup taken before the upgrade
docker compose -f docker-compose.prod.yml exec mongodb \
  mongorestore --uri="$MONGODB_URI" --archive=/backup/pre-upgrade-2026-07-18.gz --gzip --drop

# 3) Deploy the code version that matches that backup, then start
git -C /home/deploy/gococms checkout v0.9.0
docker compose -f docker-compose.prod.yml up -d gococms
```

> **Warning**
> A data restore discards everything written since the backup (posts, media, orders, audit events). Prefer code
> rollback whenever the schema is compatible; reserve data restore for genuine corruption. Object storage
> (media) and MongoDB must be restored to the **same point in time** — see [Backup & Restore](./backup-restore.md).

---

## Bare-Metal Deployment

Docker is recommended, but GOCO runs directly on a host with PHP 8.4+ and OpenSwoole 22.1+. You lose Traefik's
automation and the compose-managed backing services, so you operate MongoDB, Redis, and a reverse proxy yourself.

```bash
# 1) Install PHP 8.4 + OpenSwoole 22.1 extension, Composer, and provision
#    MongoDB (as a replica set) and Redis on the host or a private network.

# 2) Fetch and install dependencies
git clone https://github.com/gococms/gococms.git /opt/gococms
cd /opt/gococms
composer install --no-dev --optimize-autoloader

# 3) Configure environment (same keys as the Docker .env)
cp .env.example .env && chmod 600 .env   # set MONGODB_URI, REDIS_URL, secrets

# 4) Migrate + seed
php cli/goco migrate
php cli/goco db:seed

# 5) Run the ZealPHP app as a managed service (foreground: php app.php)
php app.php start -d           # start detached; also: restart | stop | status | logs
```

Run it under a supervisor for restart-on-crash and boot persistence:

```ini
# /etc/systemd/system/gococms.service
[Unit]
Description=GOCO CMS (ZealPHP / OpenSwoole)
After=network.target mongod.service redis-server.service

[Service]
User=deploy
WorkingDirectory=/opt/gococms
EnvironmentFile=/opt/gococms/.env
ExecStart=/usr/bin/php /opt/gococms/app.php
ExecReload=/bin/kill -USR1 $MAINPID       # OpenSwoole graceful worker reload
KillSignal=SIGTERM
TimeoutStopSec=30                          # drain window (matches stop_grace_period)
Restart=always

[Install]
WantedBy=multi-user.target
```

Front it with a TLS-terminating reverse proxy. You can still run [Traefik](./traefik.md) as a system service; if
you use Nginx/Apache instead, they act purely as an upstream proxy to OpenSwoole on `127.0.0.1:8080` — GOCO's
primary supported proxy remains Traefik. Graceful reload uses the same `SIGUSR1` signal as the Docker path
(`systemctl reload gococms`), and observability, secrets, upgrade, and rollback guidance above all apply
unchanged.

---

## Kubernetes Deployment

For multi-node scale, deploy the stateless app tier as a Kubernetes `Deployment` and consume MongoDB, Redis,
object storage, and search as **external managed services** (or in-cluster operators). A Helm-style layout:

```yaml
# values.yaml (excerpt) — stateless app, external stateful backends
replicaCount: 4

image:
  repository: gococms/gococms
  tag: "0.10.0"          # pin — never 'latest'
  pullPolicy: IfNotPresent

env:
  APP_ENV: production
  ZEALPHP_MODE: coroutine
  ZEALPHP_PORT: "8080"

# Backends live OUTSIDE the app pods
externalMongodb:
  uriSecret: gococms-mongodb-uri     # from a Secret / external-secrets operator
externalRedis:
  urlSecret: gococms-redis-url

# Probes map to the app's health endpoints
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
readinessProbe:
  httpGet: { path: /readyz, port: 8080 }
  initialDelaySeconds: 10

# Graceful drain — mirror OpenSwoole's stop window
terminationGracePeriodSeconds: 30

# Ingress: use Traefik as the ingress controller for parity with Docker
ingress:
  className: traefik
  hosts: [ cms.example.com, "*.example.com" ]
  tls: { enabled: true }             # cert-manager + Let's Encrypt
```

Kubernetes specifics that make GOCO behave:

- **Stateless pods only.** No persistent volumes on the app pods; state is entirely in the external Mongo/Redis/
  object-store/search services. This is what lets the `HorizontalPodAutoscaler` scale on CPU or queue depth.
- **Migrations as a pre-deploy Job/Hook.** Run `goco migrate` as a Helm pre-upgrade hook (or an Argo/Flux
  pre-sync Job) so the schema is ready before new pods roll. Because migrations are additive, a rolling
  `Deployment` update needs no downtime.
- **Rolling update strategy.** `maxUnavailable: 0`, `maxSurge: 1` plus the readiness probe gives the same
  zero-downtime behavior as the Docker rolling deploy.
- **Signals.** `terminationGracePeriodSeconds` must cover OpenSwoole's coroutine drain; Kubernetes sends
  `SIGTERM` then `SIGKILL` after the grace period, matching the `stop_grace_period` semantics above.
- **Ingress + TLS.** Use Traefik as the ingress controller with cert-manager for Let's Encrypt to keep the
  routing/middleware/security-header configuration consistent with the [Traefik](./traefik.md) doc.

> **Note**
> A first-class GOCO Helm chart is on the [Roadmap](../roadmap.md). Until it ships, the values above describe the
> contract the app expects; the same env keys, health endpoints, and drain semantics apply on any orchestrator.

---

## Production Checklist

- [ ] Exact image tag pinned in the compose/Helm values — no `latest`.
- [ ] DNS A/AAAA (and wildcard) records resolve to the host before ACME runs.
- [ ] `.env` is `chmod 600`, out of version control; secrets sourced from a manager where possible.
- [ ] `APP_ENV=production`, `APP_DEBUG=false`.
- [ ] `ACME_CA` moved from `staging` to `production`; real Let's Encrypt cert served.
- [ ] MongoDB running as a replica set (`rs0`); `MONGODB_URI` includes `replicaSet=`.
- [ ] MongoDB and Redis are **not** publicly reachable.
- [ ] `goco migrate` and `goco db:seed` completed; installer finished; `/install` disabled.
- [ ] `/healthz` and `/readyz` return `ok`; all containers `healthy`.
- [ ] 2FA/Passkeys enabled on the owner account.
- [ ] Backups scheduled and a restore test performed (see [Backup & Restore](./backup-restore.md)).
- [ ] Logs shipped off-host; metrics scraped; alerts on 5xx, p99 latency, queue depth, cert expiry.
- [ ] `stop_grace_period` / `terminationGracePeriodSeconds` ≥ 30s for graceful drain.
- [ ] Rollback path (previous tag) verified in staging.

---

## Related

- [Docker Architecture](./docker.md) — compose services, healthchecks, and image build details.
- [Traefik Reverse Proxy](./traefik.md) — auto-HTTPS, routing, middleware, and HTTP/3.
- [Backup & Restore](./backup-restore.md) — MongoDB + media backup and recovery procedures.
- [Scaling Strategy](./scaling.md) — replica sizing, autoscaling signals, and DB tiering.
- [Configuration](../getting-started/configuration.md) · [Configuration Reference](../reference/configuration-reference.md) — every environment key.
- [CLI Reference](../reference/cli-reference.md) — `goco migrate`, `db:seed`, `install`, `reload`, `logs`.
- [MongoDB Data Layer](../architecture/database-mongodb.md) · [Multi-Tenancy](../architecture/multi-tenancy.md) — data topology.
- [Authentication](../core/authentication.md) — securing the owner account after install.
- [Changelog](../changelog.md) · [Roadmap](../roadmap.md) — release notes and upcoming Helm chart.
- [Documentation Home](../README.md)
