# Configuration Reference

> The exhaustive, grouped catalogue of every GOCO CMS configuration key and environment variable — type, default, scope, reload behavior, and description — for `app.php`, `config/*.php`, `.env`, and the per-tenant `settings` collection.

This page is the **authoritative reference** for every knob GOCO CMS exposes. For the *conceptual* model — the four-layer precedence chain (package defaults → `config/*.php` → `.env` → `settings` collection), typed accessors, secrets handling, and live reload inside long-running OpenSwoole workers — read [Configuration](../getting-started/configuration.md) first. This document is the flat, alphabetized-by-area lookup you keep open while wiring a deployment.

`stable` — Keys marked `beta` / `experimental` may change before 1.0; they are flagged in-line.

---

## 1. How to Read This Reference

Every table shares the same five columns:

| Column | Meaning |
|--------|---------|
| **Key** | The environment variable (UPPER_SNAKE) and, where relevant, its `config/*.php` dotted path in parentheses. |
| **Type** | Resolved PHP type after boot-time casting: `string`, `int`, `bool` (`true/false/1/0`), `enum`, `list` (comma-separated), `duration` (seconds unless suffixed `ms`), `bytes` (accepts `512K`, `10M`, `1G`), `url`, `path`, `secret`. |
| **Default** | Value used when the key is absent. `—` means required (boot fails fast if missing in the relevant mode). |
| **Scope** | **global** = process-wide, resolved once at worker start; **workspace** = overridable per workspace/website via the `settings` collection (see [Multi-Tenancy](../architecture/multi-tenancy.md)). |
| **Description** | What it does, valid ranges, and gotchas. |

Because GOCO runs on **ZealPHP / OpenSwoole 22.1+** with long-lived workers, the environment is parsed **once at worker start**, not per request. Each key therefore has a **reload class**:

| Reload class | Symbol | How a change takes effect |
|--------------|--------|---------------------------|
| **Hot** | ♨ | Applied live — `settings`-collection (layer 4) tenant keys, and any key resolved through the settings service per request. No process action needed. |
| **Reload** | ↻ | Applied on `goco config:reload` or `SIGUSR1` (re-reads `config/*.php` and `.env`, rebuilds pools). No dropped connections. |
| **Restart** | ⟲ | Requires a full worker/process restart (`goco serve --restart` or `php app.php restart`) — anything that changes the OpenSwoole server construction: bind host/port, worker counts, socket options. |

> **Note**
> The reload symbol appears at the end of each key's description. When in doubt, a **Reload (↻)** action is always safe and never drops in-flight requests thanks to OpenSwoole graceful worker reload; only server-construction keys (⟲) need a true restart.

---

## 2. APP — Application Core

Global identity, environment, and the master secret.

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `APP_NAME` (`app.name`) | string | `GOCO CMS` | workspace | Human-readable install name; used in admin chrome, email `From` display, and `<title>` fallback. Per-website override lives in `settings`. ♨ |
| `APP_ENV` (`app.env`) | enum | `production` | global | One of `production`, `staging`, `development`, `testing`. Drives default `APP_DEBUG`, error verbosity, cache TTL floors, and whether Mailpit is auto-wired. ⟲ (changes worker/log wiring). |
| `APP_URL` (`app.url`) | url | `http://localhost:8080` | workspace | Canonical public base URL used for absolute link/asset/canonical/sitemap generation and OAuth redirect URIs. Per-website domains override this at render time. ↻ |
| `APP_KEY` (`app.key`) | secret | `—` | global | **Required.** 32-byte base64 master key (`base64:...`) for symmetric encryption of at-rest secrets and signed cookies. Generate with `goco key:generate`. Rotating it invalidates encrypted blobs — see [Security Model](../security/security-model.md). ↻ |
| `APP_DEBUG` (`app.debug`) | bool | `false` | global | When `true`, returns stack traces in responses and disables view caching. **Never** `true` in production. Defaults to `true` only when `APP_ENV=development`. ↻ |
| `APP_LOCALE` (`app.locale`) | string | `en` | workspace | Default BCP-47 locale for i18n and date/number formatting. ♨ |
| `APP_FALLBACK_LOCALE` (`app.fallback_locale`) | string | `en` | workspace | Locale used when a translation key is missing in the active locale. ♨ |
| `APP_TIMEZONE` (`app.timezone`) | string | `UTC` | workspace | IANA timezone for scheduling, cron display, and audit-log rendering. Storage is always UTC in MongoDB. ♨ |
| `APP_MAINTENANCE` (`app.maintenance`) | bool | `false` | workspace | Serves a 503 maintenance page to non-privileged users while keeping the admin reachable. ♨ |
| `APP_INSTALLED` (`app.installed`) | bool | `false` | global | Set `true` by the installer once bootstrap collections exist; when `false`, all traffic routes to `apps/installer`. ↻ |

```env
APP_NAME="GOCO CMS"
APP_ENV=production
APP_URL=https://cms.example.com
APP_KEY=base64:Zx8s...==
APP_DEBUG=false
APP_TIMEZONE=UTC
```

---

## 3. SERVER — ZealPHP / OpenSwoole Runtime

These construct the OpenSwoole HTTP server in `app.php` (`$app = App::init($host, $port)`). Almost all are **Restart (⟲)** because they change how the server is built.

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `SERVER_HOST` (`server.host`) | string | `0.0.0.0` | global | Bind address passed to `App::init()`. Inside Docker keep `0.0.0.0` so Traefik can reach the container. ⟲ |
| `SERVER_PORT` (`server.port`) | int | `8080` | global | Bind port passed to `App::init()`. Traefik targets this via the service loadbalancer label. ⟲ |
| `SERVER_MODE` (`server.mode`) | enum | `coroutine` | global | Maps to `App::mode()`: `coroutine` → `MODE_COROUTINE` (modern default), `legacy_cgi` → `MODE_LEGACY_CGI`, `coroutine_legacy` → `MODE_COROUTINE_LEGACY`, `mixed` → `MODE_MIXED`. ⟲ |
| `SERVER_WORKERS` (`server.workers`) | int | `#CPU cores` | global | OpenSwoole `worker_num`. Each worker is a coroutine scheduler; size to CPU, not to concurrency (coroutines handle concurrency). ⟲ |
| `SERVER_TASK_WORKERS` (`server.task_workers`) | int | `2 × #CPU` | global | `task_worker_num` for offloaded blocking work dispatched via the task API. `0` disables tasking. ⟲ |
| `SERVER_MAX_REQUESTS` (`server.max_requests`) | int | `0` | global | Recycle a worker after N requests to bound memory leakage in long-running processes. `0` = never recycle. ⟲ |
| `SERVER_MAX_CONN` (`server.max_connections`) | int | `10000` | global | OpenSwoole `max_connection` ceiling. Must be ≤ OS file-descriptor limit. ⟲ |
| `SERVER_BUFFER_OUTPUT` (`server.buffer_output_size`) | bytes | `32M` | global | Max buffered response size per connection; raise for large streamed exports. ⟲ |
| `SERVER_PACKAGE_MAX` (`server.package_max_length`) | bytes | `16M` | global | Max inbound request/WebSocket frame; align with `SECURITY_MAX_UPLOAD`. ⟲ |
| `SERVER_ENABLE_COROUTINE` (`server.enable_coroutine`) | bool | `true` | global | Wrap each request in a coroutine. Keep `true` unless running pure `MODE_LEGACY_CGI`. ⟲ |
| `SERVER_HTTP_COMPRESSION` (`server.http_compression`) | bool | `true` | global | OpenSwoole gzip/br for responses (in addition to the Compression middleware). ↻ |
| `SERVER_GRACEFUL_TIMEOUT` (`server.graceful_timeout`) | duration | `30` | global | Seconds a worker may finish in-flight coroutines during graceful shutdown/reload. ⟲ |
| `SERVER_PID_FILE` (`server.pid_file`) | path | `/tmp/zealphp/goco.pid` | global | PID file for `start -d` / `restart` / `stop` / `status`. ⟲ |
| `SERVER_LOG_DIR` (`server.log_dir`) | path | `/tmp/zealphp/` | global | Directory for ZealPHP process logs (`goco logs` tails these). ↻ |
| `SERVER_LOG_LEVEL` (`server.log_level`) | enum | `info` | global | `debug`, `info`, `notice`, `warning`, `error`. ↻ |

> **Warning**
> `SERVER_WORKERS` and `SERVER_TASK_WORKERS` are the two most impactful tuning knobs. Because OpenSwoole workers are long-lived and share nothing but `\ZealPHP\Store` (OpenSwoole\Table) and Redis, oversizing workers wastes RAM without improving throughput. Start at CPU-core count and load-test. See [Scaling Strategy](../deployment/scaling.md).

```env
SERVER_HOST=0.0.0.0
SERVER_PORT=8080
SERVER_MODE=coroutine
SERVER_WORKERS=8
SERVER_TASK_WORKERS=16
SERVER_MAX_REQUESTS=100000
```

---

## 4. MONGODB — Primary Database

The single logical database per deployment. See [MongoDB Data Layer](../architecture/database-mongodb.md).

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `MONGODB_URI` (`mongodb.uri`) | secret | `mongodb://mongodb:27017` | global | **Required.** Standard connection string (`mongodb://` or `mongodb+srv://`). May embed credentials, replica set, and read-preference; prefer discrete keys below over embedding. ↻ |
| `MONGODB_DB` (`mongodb.database`) | string | `goco` | global | Logical database name holding all collections for this deployment. ↻ |
| `MONGODB_USERNAME` (`mongodb.username`) | string | `null` | global | Auth user (overrides any user in the URI). ↻ |
| `MONGODB_PASSWORD` (`mongodb.password`) | secret | `null` | global | Auth password. Store via Docker/K8s secret, never commit. ↻ |
| `MONGODB_AUTH_SOURCE` (`mongodb.auth_source`) | string | `admin` | global | Authentication database. ↻ |
| `MONGODB_REPLICA_SET` (`mongodb.replica_set`) | string | `null` | global | Replica-set name; **required** to enable multi-document transactions used for cross-collection invariants. ↻ |
| `MONGODB_READ_PREFERENCE` (`mongodb.read_preference`) | enum | `primaryPreferred` | global | `primary`, `primaryPreferred`, `secondary`, `secondaryPreferred`, `nearest`. ↻ |
| `MONGODB_WRITE_CONCERN` (`mongodb.write_concern`) | string | `majority` | global | Write concern `w`; `majority` recommended for durability with replica sets. ↻ |
| `MONGODB_POOL_SIZE` (`mongodb.pool_size`) | int | `16` | global | Coroutine-aware connection-pool size **per worker**. Effective connections ≈ `SERVER_WORKERS × MONGODB_POOL_SIZE`; keep under the server's `maxIncomingConnections`. ↻ |
| `MONGODB_POOL_MIN` (`mongodb.pool_min`) | int | `2` | global | Warm connections held open per worker even when idle. ↻ |
| `MONGODB_CONNECT_TIMEOUT` (`mongodb.connect_timeout_ms`) | duration(ms) | `5000ms` | global | Socket connect timeout. ↻ |
| `MONGODB_QUERY_TIMEOUT` (`mongodb.query_timeout_ms`) | duration(ms) | `15000ms` | global | Server-side `maxTimeMS` applied to reads/aggregations. ↻ |
| `MONGODB_TLS` (`mongodb.tls`) | bool | `false` | global | Enable TLS to MongoDB (Atlas/managed). ↻ |
| `MONGODB_TLS_CA_FILE` (`mongodb.tls_ca_file`) | path | `null` | global | CA bundle path when using private CAs. ↻ |
| `MONGODB_RETRY_WRITES` (`mongodb.retry_writes`) | bool | `true` | global | Enable retryable writes (requires replica set). ↻ |

> **Tip**
> Multi-document transactions, soft-delete + versioning invariants, and audit-log co-writes require a **replica set**. A standalone `mongod` boots fine for development but silently degrades transactional guarantees — set `MONGODB_REPLICA_SET` in every non-trivial environment.

---

## 5. REDIS — Cache, Queue, Sessions, Realtime, Locks

One Redis powers cache, queue, sessions, rate limiting, distributed locks, and pub/sub. See [Caching, Queue & Realtime](../architecture/caching-and-queue.md).

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `REDIS_URL` (`redis.url`) | secret | `redis://redis:6379` | global | **Required.** Full DSN (`redis://`, `rediss://`, or `redis+sentinel://`). Overrides the discrete host/port/password keys when set. ↻ |
| `REDIS_HOST` (`redis.host`) | string | `redis` | global | Host (used when `REDIS_URL` is unset). ↻ |
| `REDIS_PORT` (`redis.port`) | int | `6379` | global | Port. ↻ |
| `REDIS_PASSWORD` (`redis.password`) | secret | `null` | global | AUTH password. ↻ |
| `REDIS_DB` (`redis.database`) | int | `0` | global | Logical DB index. ↻ |
| `REDIS_PREFIX` (`redis.prefix`) | string | `goco:` | global | Namespace prefix on every key; isolate multiple installs sharing one Redis. ↻ |
| `REDIS_POOL_SIZE` (`redis.pool_size`) | int | `32` | global | Coroutine connection-pool size per worker. ↻ |
| `REDIS_TLS` (`redis.tls`) | bool | `false` | global | Use `rediss://` TLS. ↻ |
| `REDIS_TIMEOUT` (`redis.timeout_ms`) | duration(ms) | `2000ms` | global | Command timeout. ↻ |

### 5.1 CACHE

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `CACHE_DRIVER` (`cache.driver`) | enum | `redis` | global | `redis` (shared, default), `table` (per-worker `\ZealPHP\Store`/OpenSwoole\Table, ultra-fast but worker-local), `array` (per-request, testing), `null`. ↻ |
| `CACHE_PREFIX` (`cache.prefix`) | string | `goco:cache:` | global | Key prefix for cache entries. ↻ |
| `CACHE_DEFAULT_TTL` (`cache.default_ttl`) | duration | `3600` | workspace | Default TTL for `remember()`-style entries. ♨ |
| `CACHE_PAGE_TTL` (`cache.page_ttl`) | duration | `300` | workspace | Full-page/fragment cache TTL for the rendering pipeline. ♨ |
| `CACHE_STORE_BACKEND` (`cache.store_backend`) | enum | `table` | global | Backend for `\ZealPHP\Store`: `table` (worker-local) or `redis` (`Store::defaultBackend(Store::BACKEND_REDIS)`, cross-worker). ↻ |

### 5.2 QUEUE

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `QUEUE_DRIVER` (`queue.driver`) | enum | `redis` | global | `redis` (default), `sync` (inline, testing), `null`. ↻ |
| `QUEUE_DEFAULT` (`queue.default`) | string | `default` | global | Default queue name for dispatched jobs. ↻ |
| `QUEUE_PREFIX` (`queue.prefix`) | string | `goco:queue:` | global | Redis key prefix for queues. ↻ |
| `QUEUE_RETRY_AFTER` (`queue.retry_after`) | duration | `90` | global | Seconds before a reserved job is considered failed and retried. Must exceed the longest job runtime. ↻ |
| `QUEUE_MAX_ATTEMPTS` (`queue.max_attempts`) | int | `3` | workspace | Attempts before a job moves to the dead-letter set and a `jobs` doc is marked `failed`. ♨ |
| `QUEUE_BACKOFF` (`queue.backoff`) | list | `10,60,300` | workspace | Per-attempt backoff seconds. ♨ |
| `QUEUE_WORKERS` (`queue.workers`) | int | `4` | global | Coroutine consumers per OpenSwoole worker (started in `App::onWorkerStart`). ⟲ |
| `QUEUE_SCHEDULER` (`queue.scheduler_enabled`) | bool | `true` | global | Enable the `App::tick`-driven cron scheduler for recurring jobs. ⟲ |

---

## 6. SESSION

Redis-backed, per-coroutine-isolated sessions (`$_SESSION` overridden by ext-zealphp). See [Authentication](../core/authentication.md).

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `SESSION_DRIVER` (`session.driver`) | enum | `redis` | global | `redis` (default, shared across workers), `cookie` (encrypted, stateless), `null`. ↻ |
| `SESSION_PREFIX` (`session.prefix`) | string | `goco:sess:` | global | Redis key prefix for session blobs. ↻ |
| `SESSION_LIFETIME` (`session.lifetime`) | duration | `7200` | workspace | Idle lifetime in seconds; sliding on activity. ♨ |
| `SESSION_COOKIE` (`session.cookie`) | string | `goco_session` | global | Session cookie name. ↻ |
| `SESSION_COOKIE_SECURE` (`session.secure`) | bool | `true` | global | `Secure` flag; keep `true` behind Traefik HTTPS. ↻ |
| `SESSION_COOKIE_HTTPONLY` (`session.http_only`) | bool | `true` | global | `HttpOnly` flag. ↻ |
| `SESSION_COOKIE_SAMESITE` (`session.same_site`) | enum | `lax` | global | `lax`, `strict`, or `none` (requires Secure). ↻ |
| `SESSION_DOMAIN` (`session.domain`) | string | `null` | workspace | Cookie domain; set for shared-cookie multi-domain setups. ↻ |

---

## 7. STORAGE — Object & Media

Driver-based storage: **Local**, **MinIO**, **Amazon S3**. See [Storage & Media](../architecture/storage.md).

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `STORAGE_DRIVER` (`storage.driver`) | enum | `local` | workspace | `local`, `minio`, `s3`. Selects the active driver; per-workspace override lets tenants use isolated buckets. ↻ |
| `STORAGE_URL` (`storage.url`) | url | `{APP_URL}/storage` | workspace | Public base URL for served assets/CDN in front of the driver. ♨ |
| `STORAGE_MAX_UPLOAD` (`storage.max_upload`) | bytes | `25M` | workspace | Per-file upload ceiling enforced by the media service; keep ≤ `SERVER_PACKAGE_MAX`. ♨ |
| `STORAGE_VISIBILITY` (`storage.visibility`) | enum | `private` | workspace | Default object ACL: `private` (signed URLs) or `public`. ♨ |

### 7.1 Local driver

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `STORAGE_LOCAL_ROOT` (`storage.local.root`) | path | `storage/app` | global | Filesystem root for the local driver. Mount as a Docker volume for persistence. ↻ |
| `STORAGE_LOCAL_PERMISSIONS` (`storage.local.permissions`) | string | `0640` | global | Octal file mode for written objects. ↻ |

### 7.2 MinIO / S3-compatible driver

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `STORAGE_S3_KEY` (`storage.s3.key`) | secret | `—` | workspace | Access key ID (required when driver = `minio`/`s3`). ↻ |
| `STORAGE_S3_SECRET` (`storage.s3.secret`) | secret | `—` | workspace | Secret access key. ↻ |
| `STORAGE_S3_REGION` (`storage.s3.region`) | string | `us-east-1` | workspace | Region; MinIO accepts any value. ↻ |
| `STORAGE_S3_BUCKET` (`storage.s3.bucket`) | string | `—` | workspace | Target bucket. ↻ |
| `STORAGE_S3_ENDPOINT` (`storage.s3.endpoint`) | url | `null` | workspace | Custom endpoint — set to the MinIO URL (e.g. `http://minio:9000`); leave null for real AWS S3. ↻ |
| `STORAGE_S3_PATH_STYLE` (`storage.s3.use_path_style`) | bool | `false` | workspace | Path-style addressing; **`true` for MinIO**, `false` for AWS. ↻ |
| `STORAGE_S3_URL_EXPIRY` (`storage.s3.signed_url_ttl`) | duration | `900` | workspace | TTL for presigned URLs of private objects. ♨ |

```env
STORAGE_DRIVER=minio
STORAGE_S3_KEY=minioadmin
STORAGE_S3_SECRET=minioadmin
STORAGE_S3_BUCKET=goco-media
STORAGE_S3_ENDPOINT=http://minio:9000
STORAGE_S3_PATH_STYLE=true
```

---

## 8. SEARCH

Swappable provider interface: **MongoDB text/Atlas Search**, **Meilisearch**, **OpenSearch**. See [Search](../architecture/search.md).

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `SEARCH_PROVIDER` (`search.provider`) | enum | `mongodb` | workspace | `mongodb` (text index / Atlas Search, zero extra infra), `meilisearch`, `opensearch`. ↻ |
| `SEARCH_INDEX_PREFIX` (`search.index_prefix`) | string | `goco_` | global | Prefix for external index names; isolates tenants/installs. ↻ |
| `SEARCH_BATCH_SIZE` (`search.batch_size`) | int | `500` | global | Documents per indexing batch during reindex jobs. ♨ |
| `SEARCH_QUEUE_INDEXING` (`search.queue_indexing`) | bool | `true` | workspace | Index asynchronously via the queue instead of on the write path. ♨ |

### 8.1 MongoDB provider

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `SEARCH_MONGODB_ATLAS` (`search.mongodb.atlas`) | bool | `false` | global | Use Atlas Search (`$search`) instead of a classic `$text` index. ↻ |
| `SEARCH_MONGODB_INDEX` (`search.mongodb.atlas_index`) | string | `default` | global | Atlas Search index name (when `atlas=true`). ↻ |

### 8.2 Meilisearch provider

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `MEILISEARCH_HOST` (`search.meilisearch.host`) | url | `http://meilisearch:7700` | global | Meilisearch endpoint. ↻ |
| `MEILISEARCH_KEY` (`search.meilisearch.key`) | secret | `—` | global | Master/API key (required when provider = `meilisearch`). ↻ |
| `MEILISEARCH_TIMEOUT` (`search.meilisearch.timeout_ms`) | duration(ms) | `3000ms` | global | Request timeout. ↻ |

### 8.3 OpenSearch provider

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `OPENSEARCH_HOSTS` (`search.opensearch.hosts`) | list | `https://opensearch:9200` | global | Comma-separated node URLs. ↻ |
| `OPENSEARCH_USERNAME` (`search.opensearch.username`) | string | `admin` | global | Basic-auth user. ↻ |
| `OPENSEARCH_PASSWORD` (`search.opensearch.password`) | secret | `—` | global | Basic-auth password. ↻ |
| `OPENSEARCH_VERIFY_TLS` (`search.opensearch.verify_tls`) | bool | `true` | global | Verify node TLS certs; disable only for self-signed dev clusters. ↻ |

---

## 9. MAIL

Dev delivery via **Mailpit**; production via SMTP. See [Configuration](../getting-started/configuration.md).

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `MAIL_DRIVER` (`mail.driver`) | enum | `smtp` | workspace | `smtp`, `mailpit` (alias targeting the dev container), `log` (write to log), `null`. In `APP_ENV=development` defaults to `mailpit`. ↻ |
| `MAIL_HOST` (`mail.host`) | string | `mailpit` | global | SMTP host. For the bundled dev container use `mailpit`. ↻ |
| `MAIL_PORT` (`mail.port`) | int | `1025` | global | SMTP port (Mailpit listens on `1025`; its web UI on `8025`). ↻ |
| `MAIL_USERNAME` (`mail.username`) | string | `null` | global | SMTP auth user. ↻ |
| `MAIL_PASSWORD` (`mail.password`) | secret | `null` | global | SMTP auth password. ↻ |
| `MAIL_ENCRYPTION` (`mail.encryption`) | enum | `null` | global | `tls`, `ssl`, or `null` (Mailpit needs none). ↻ |
| `MAIL_FROM_ADDRESS` (`mail.from.address`) | string | `hello@example.com` | workspace | Default envelope/`From` address. ♨ |
| `MAIL_FROM_NAME` (`mail.from.name`) | string | `{APP_NAME}` | workspace | Default `From` display name. ♨ |
| `MAIL_QUEUE` (`mail.queue`) | bool | `true` | workspace | Send via the queue rather than on the request path. ♨ |

---

## 10. TRAEFIK, DOMAINS & TLS/ACME

GOCO is Docker-first behind **Traefik** (auto HTTPS, HTTP/3, wildcard/multi-domain). Most keys here are consumed by Docker labels and the `docker/` compose files rather than by the PHP process; they are listed because they belong to the deployment configuration surface. See [Traefik Reverse Proxy](../deployment/traefik.md) and [Docker Architecture](../deployment/docker.md).

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `TRAEFIK_DOMAIN` (`traefik.domain`) | string | `localhost` | global | Primary/apex domain for the root router and the Traefik dashboard host. ⟲ (recreate container). |
| `TRAEFIK_WILDCARD` (`traefik.wildcard`) | bool | `false` | global | Enable a wildcard router (`*.{TRAEFIK_DOMAIN}`) for per-tenant subdomains — requires DNS-01 ACME. ⟲ |
| `TRAEFIK_ENTRYPOINT_WEB` (`traefik.entrypoint.web`) | string | `web` | global | Name of the HTTP (`:80`) entrypoint; redirects to `websecure`. ⟲ |
| `TRAEFIK_ENTRYPOINT_WEBSECURE` (`traefik.entrypoint.websecure`) | string | `websecure` | global | Name of the HTTPS (`:443`) entrypoint. ⟲ |
| `TRAEFIK_HTTP3` (`traefik.http3`) | bool | `true` | global | Advertise and serve HTTP/3 (QUIC, UDP/443) on the secure entrypoint. ⟲ |
| `TRAEFIK_DASHBOARD` (`traefik.dashboard`) | bool | `false` | global | Expose the Traefik dashboard (protect with BasicAuth middleware). ⟲ |
| `ACME_EMAIL` (`traefik.acme.email`) | string | `—` | global | **Required for HTTPS.** Contact email for Let's Encrypt registration and expiry notices. ⟲ |
| `ACME_CA_SERVER` (`traefik.acme.ca_server`) | url | `https://acme-v02.api.letsencrypt.org/directory` | global | ACME directory; point at the staging URL while testing to avoid rate limits. ⟲ |
| `ACME_CHALLENGE` (`traefik.acme.challenge`) | enum | `tls` | global | `tls` (TLS-ALPN-01), `http` (HTTP-01), or `dns` (DNS-01, required for wildcard). ⟲ |
| `ACME_DNS_PROVIDER` (`traefik.acme.dns_provider`) | string | `null` | global | Lego DNS provider code (e.g. `cloudflare`, `route53`) when `ACME_CHALLENGE=dns`. ⟲ |
| `ACME_STORAGE` (`traefik.acme.storage`) | path | `/letsencrypt/acme.json` | global | Certificate store path (chmod `0600`, mounted volume). ⟲ |
| `TRAEFIK_RATE_LIMIT_AVG` (`traefik.rate_limit.average`) | int | `100` | global | Edge rate-limit middleware: average requests/sec per source. ⟲ |
| `TRAEFIK_RATE_LIMIT_BURST` (`traefik.rate_limit.burst`) | int | `200` | global | Edge rate-limit burst allowance. ⟲ |

The application itself resolves the incoming host to a tenant using the **`domains`** collection, not env vars — Traefik terminates TLS and forwards to `gococms:{SERVER_PORT}`; GOCO maps `Host` → `website_id`. See [Multi-Tenancy](../architecture/multi-tenancy.md).

> **Note**
> Wildcard certificates (`*.example.com`) require the **DNS-01** challenge, so `ACME_CHALLENGE=dns` and a configured `ACME_DNS_PROVIDER` (plus that provider's own API credentials) are mandatory when `TRAEFIK_WILDCARD=true`.

---

## 11. AI PLATFORM

Provider-agnostic AI with per-workspace keys, budgets, and rate limits. See [AI Platform](../core/ai-platform.md).

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `AI_ENABLED` (`ai.enabled`) | bool | `false` | workspace | Master switch for AI features (content assist, alt-text, embeddings, semantic search). ♨ |
| `AI_PROVIDER` (`ai.provider`) | enum | `anthropic` | workspace | `anthropic`, `openai`, `azure-openai`, `ollama`, `custom`. Selects the active driver. ↻ |
| `AI_API_KEY` (`ai.api_key`) | secret | `—` | workspace | Provider API key (required when `AI_ENABLED=true` and provider needs auth). Stored encrypted with `APP_KEY` when set per-workspace. ♨ |
| `AI_BASE_URL` (`ai.base_url`) | url | `null` | workspace | Custom endpoint for `azure-openai`, `ollama`, or `custom` gateways. ♨ |
| `AI_DEFAULT_MODEL` (`ai.default_model`) | string | `provider default` | workspace | Model id used when a call omits one (e.g. a text model for content assist). ♨ |
| `AI_EMBEDDING_MODEL` (`ai.embedding_model`) | string | `provider default` | workspace | Model id for vector embeddings used by semantic search. ♨ |
| `AI_MAX_TOKENS` (`ai.max_tokens`) | int | `2048` | workspace | Default max output tokens per request. ♨ |
| `AI_TIMEOUT` (`ai.timeout_ms`) | duration(ms) | `30000ms` | workspace | Per-request timeout. ♨ |
| `AI_MONTHLY_BUDGET` (`ai.monthly_budget`) | int | `0` | workspace | Soft monthly spend cap (provider currency units, ×100 for cents-precision); `0` = unlimited. Enforced by the AI meter. ♨ |
| `AI_RATE_LIMIT` (`ai.rate_limit_per_min`) | int | `60` | workspace | Requests/minute per workspace, enforced via Redis. ♨ |
| `AI_LOG_PROMPTS` (`ai.log_prompts`) | bool | `false` | workspace | Persist prompt/response pairs to `audit_logs` for debugging (privacy-sensitive — off by default). ♨ |

> **Warning**
> AI keys are **workspace-scoped secrets**. When set through the admin they are encrypted at rest with `APP_KEY`; an env-level `AI_API_KEY` acts only as the global fallback. Rotating `APP_KEY` requires re-entering per-workspace AI keys.

---

## 12. SECURITY

CSRF, JWT, password hashing, and application-layer rate limits. See [Security Model](../security/security-model.md) and [Authentication](../core/authentication.md).

### 12.1 CSRF & headers

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `SECURITY_CSRF_ENABLED` (`security.csrf.enabled`) | bool | `true` | global | Enable the ZealPHP `Csrf` middleware for state-changing routes. ↻ |
| `SECURITY_CSRF_TTL` (`security.csrf.ttl`) | duration | `7200` | global | Lifetime of a CSRF token. ↻ |
| `SECURITY_HSTS` (`security.hsts`) | bool | `true` | global | Emit `Strict-Transport-Security` (via app + Traefik headers middleware). ↻ |
| `SECURITY_CSP` (`security.csp`) | string | `default-src 'self'` | workspace | Content-Security-Policy header value. ♨ |
| `SECURITY_MAX_UPLOAD` (`security.max_upload`) | bytes | `25M` | workspace | Hard request-body limit enforced by middleware; coordinate with `STORAGE_MAX_UPLOAD` and `SERVER_PACKAGE_MAX`. ♨ |

### 12.2 JWT (API tokens)

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `JWT_SECRET` (`security.jwt.secret`) | secret | `{APP_KEY}` | global | HMAC signing secret for API JWTs. Defaults to `APP_KEY`; set separately to rotate API tokens without touching data encryption. ↻ |
| `JWT_ALGO` (`security.jwt.algo`) | enum | `HS256` | global | `HS256`, `HS384`, `HS512`, or `RS256` (with key files below). ↻ |
| `JWT_TTL` (`security.jwt.ttl`) | duration | `900` | global | Access-token lifetime (15 min). ↻ |
| `JWT_REFRESH_TTL` (`security.jwt.refresh_ttl`) | duration | `1209600` | global | Refresh-token lifetime (14 days); refresh tokens tracked in Redis for revocation. ↻ |
| `JWT_ISSUER` (`security.jwt.issuer`) | string | `{APP_URL}` | global | `iss` claim. ↻ |
| `JWT_PRIVATE_KEY` (`security.jwt.private_key`) | path | `null` | global | PEM private key path (required for `RS256`). ↻ |
| `JWT_PUBLIC_KEY` (`security.jwt.public_key`) | path | `null` | global | PEM public key path (`RS256` verification). ↻ |

### 12.3 Password hashing (Argon2id)

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `ARGON_MEMORY` (`security.argon.memory_cost`) | int | `65536` | global | Argon2id memory cost in KiB (64 MiB). Raise for stronger hashing at higher CPU/RAM cost. ↻ |
| `ARGON_TIME` (`security.argon.time_cost`) | int | `4` | global | Argon2id iterations (time cost). ↻ |
| `ARGON_THREADS` (`security.argon.threads`) | int | `2` | global | Argon2id parallelism (lanes). ↻ |

### 12.4 Authentication features

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `AUTH_2FA_ENABLED` (`security.auth.totp`) | bool | `true` | workspace | Allow TOTP-based 2FA enrollment. ♨ |
| `AUTH_PASSKEYS_ENABLED` (`security.auth.passkeys`) | bool | `true` | workspace | Allow WebAuthn passkey registration/login. ♨ |
| `AUTH_PASSKEY_RP_ID` (`security.auth.rp_id`) | string | `{host of APP_URL}` | workspace | WebAuthn Relying-Party ID (usually the apex domain). ♨ |
| `AUTH_OAUTH_ENABLED` (`security.auth.oauth`) | bool | `false` | workspace | Enable OAuth2 social/SSO login providers configured in `settings`. ♨ |
| `AUTH_LOCKOUT_ATTEMPTS` (`security.auth.lockout_attempts`) | int | `5` | workspace | Failed logins before temporary lockout. ♨ |
| `AUTH_LOCKOUT_DECAY` (`security.auth.lockout_decay`) | duration | `900` | workspace | Lockout window in seconds. ♨ |

### 12.5 Application rate limiting

| Key | Type | Default | Scope | Description |
|-----|------|---------|-------|-------------|
| `RATE_LIMIT_ENABLED` (`security.rate_limit.enabled`) | bool | `true` | global | Enable the ZealPHP `RateLimit` middleware (Redis-backed). ↻ |
| `RATE_LIMIT_WEB` (`security.rate_limit.web`) | int | `120` | workspace | Requests/minute per client for web routes. ♨ |
| `RATE_LIMIT_API` (`security.rate_limit.api`) | int | `600` | workspace | Requests/minute per API token. ♨ |
| `RATE_LIMIT_LOGIN` (`security.rate_limit.login`) | int | `10` | workspace | Login attempts/minute per IP. ♨ |
| `CONCURRENCY_LIMIT` (`security.concurrency_limit`) | int | `0` | global | Max concurrent in-flight requests via the `ConcurrencyLimit` middleware; `0` = unlimited. ↻ |

---

## 13. Reload Behavior Summary

| Reload class | When to use | Command |
|--------------|-------------|---------|
| **Hot (♨)** | `settings`-collection / workspace-scoped keys | None — applied on next request |
| **Reload (↻)** | `config/*.php` and `.env` values not tied to server construction (DB/Redis/mail/search endpoints, secrets, timeouts, TTLs) | `goco config:reload` or `kill -USR1 <pid>` |
| **Restart (⟲)** | Server construction: `SERVER_HOST/PORT/MODE/WORKERS/TASK_WORKERS`, queue workers, and Traefik/ACME (recreate container) | `php app.php restart` / `goco serve --restart` / `docker compose up -d` |

```bash
# Hot reload of config/*.php + .env without dropping connections
goco config:reload

# Full restart (server-construction changes)
php app.php restart          # or: docker compose restart gococms

# Verify what the running process resolved
goco config:show --resolved
goco config:show storage     # one group
```

> **Tip**
> After editing `.env`, run `goco config:check` before reloading — it validates types (e.g. a non-integer `MONGODB_POOL_SIZE`), asserts required keys are present for the active drivers, and warns when `APP_DEBUG=true` under `APP_ENV=production`.

---

## 14. Minimal Production `.env`

A complete, non-placeholder baseline for a Docker + Traefik deployment:

```env
# APP
APP_NAME="GOCO CMS"
APP_ENV=production
APP_URL=https://cms.example.com
APP_KEY=base64:REPLACE_WITH_goco_key_generate
APP_DEBUG=false
APP_TIMEZONE=UTC

# SERVER
SERVER_HOST=0.0.0.0
SERVER_PORT=8080
SERVER_MODE=coroutine
SERVER_WORKERS=8
SERVER_TASK_WORKERS=16

# MONGODB
MONGODB_URI=mongodb://mongodb:27017
MONGODB_DB=goco
MONGODB_REPLICA_SET=rs0
MONGODB_POOL_SIZE=16

# REDIS
REDIS_URL=redis://redis:6379
REDIS_PREFIX=goco:

# CACHE / QUEUE / SESSION
CACHE_DRIVER=redis
QUEUE_DRIVER=redis
SESSION_DRIVER=redis

# STORAGE
STORAGE_DRIVER=minio
STORAGE_S3_KEY=minioadmin
STORAGE_S3_SECRET=REPLACE_ME
STORAGE_S3_BUCKET=goco-media
STORAGE_S3_ENDPOINT=http://minio:9000
STORAGE_S3_PATH_STYLE=true

# SEARCH
SEARCH_PROVIDER=meilisearch
MEILISEARCH_HOST=http://meilisearch:7700
MEILISEARCH_KEY=REPLACE_ME

# MAIL
MAIL_DRIVER=smtp
MAIL_HOST=smtp.example.com
MAIL_PORT=587
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=hello@example.com

# TRAEFIK / TLS
TRAEFIK_DOMAIN=example.com
TRAEFIK_WILDCARD=true
TRAEFIK_HTTP3=true
ACME_EMAIL=ops@example.com
ACME_CHALLENGE=dns
ACME_DNS_PROVIDER=cloudflare

# SECURITY
JWT_TTL=900
JWT_REFRESH_TTL=1209600
RATE_LIMIT_API=600

# AI (optional)
AI_ENABLED=false
AI_PROVIDER=anthropic
```

> **Warning**
> Never commit a populated `.env`. Every `secret`-typed key belongs in a Docker/Kubernetes secret or a vault, injected at runtime. `APP_KEY`, `MONGODB_PASSWORD`, `REDIS_PASSWORD`, `STORAGE_S3_SECRET`, `MEILISEARCH_KEY`, `JWT_SECRET`, and `AI_API_KEY` must be treated as production credentials.

---

## Related

- [Configuration](../getting-started/configuration.md) — the layering model, precedence, and reload internals
- [Installation](../getting-started/installation.md) — initial `.env` scaffolding via the installer
- [Configuration Reference is complemented by the CLI Reference](../reference/cli-reference.md) — `goco config:*`, `key:generate`, `serve`
- [API Reference](../reference/api-reference.md) — endpoints affected by JWT/rate-limit keys
- [MongoDB Data Layer](../architecture/database-mongodb.md) — pooling, transactions, replica sets
- [Caching, Queue & Realtime (Redis)](../architecture/caching-and-queue.md) — cache/queue/session drivers
- [Storage & Media](../architecture/storage.md) — Local/MinIO/S3 drivers
- [Search](../architecture/search.md) — MongoDB/Meilisearch/OpenSearch providers
- [Multi-Tenancy](../architecture/multi-tenancy.md) — workspace-scoped settings and domain resolution
- [Security Model](../security/security-model.md) — CSRF, JWT, Argon2id, rate limiting
- [Traefik Reverse Proxy](../deployment/traefik.md) & [Docker Architecture](../deployment/docker.md) — TLS/ACME and service wiring
- [Documentation Index](../README.md)
