# Configuration

> How GOCO CMS resolves configuration — from framework defaults through `config/*.php`, `.env`, and the per-workspace `settings` collection — plus secrets handling and live reload inside long-running OpenSwoole workers.

GOCO CMS follows a strict **12-factor** configuration model: everything that varies between deployments (credentials, hostnames, feature flags, service endpoints) lives in the environment, while everything structural (schemas, defaults, wiring) lives in versioned code. Because GOCO runs on **ZealPHP / OpenSwoole**, workers are long-lived — configuration is parsed **once at worker start** and held in memory, not re-read on every request like a classic PHP-FPM app. This makes config resolution fast, but it also means changes require an explicit reload path, described in [§8](#8-reloading-config-in-a-long-running-worker).

This page explains the *layering model* and walks the key groups with real examples. For the exhaustive, alphabetized list of every key, its type, and its default, see the [Configuration Reference](../reference/configuration-reference.md).

---

## 1. Configuration Philosophy

GOCO's configuration obeys four principles:

1. **Env-driven (12-factor).** No secret, hostname, or endpoint is ever committed to git. The `.env` file (git-ignored) and the process environment are the *only* sources of deployment-specific values. Code reads them through a typed `config()` accessor, never `getenv()` directly.
2. **Layered with clear precedence.** Configuration is composed from four layers that override one another in a fixed order (see [§2](#2-the-four-layers)). A missing value falls through to the next-lower layer, ending at a hard-coded default so the system always boots.
3. **Typed and validated at boot.** Every `config/*.php` file returns a plain PHP array. At worker start these are merged, type-cast, and validated; a malformed value (e.g. a non-integer `REDIS_POOL_SIZE`) fails fast with a descriptive error rather than surfacing as a runtime bug.
4. **Tenant-aware.** Infrastructure config (DB, Redis, storage) is process-global. Product config (site name, locale, SEO defaults, mail-from, feature toggles) can be **overridden per workspace and per website** via the `settings` collection, so one running process serves many tenants with different behavior. See [Multi-Tenancy](../architecture/multi-tenancy.md).

> **Note**
> GOCO never mutates process global state for config. `App::superglobals(false)` is set in `app.php`, and per-request/per-coroutine values (locale, active website, resolved tenant) live in [`Goco\G` / RequestContext](../architecture/request-lifecycle.md), never in `$_ENV`.

---

## 2. The Four Layers

Configuration is resolved by merging four layers. **Later layers win.** Higher-numbered layers below override lower ones for the same key.

```mermaid
flowchart LR
    A["1. Framework &<br/>package defaults<br/><i>(hard-coded)</i>"] --> B["2. config/*.php<br/><i>(versioned, typed)</i>"]
    B --> C["3. .env / process env<br/><i>(git-ignored secrets)</i>"]
    C --> D["4. settings collection<br/><i>(per-workspace / per-website)</i>"]
    D --> E([Effective config])
    style E fill:#0b7285,stroke:#083f4d,color:#fff
```

| Layer | Source | Scope | Mutable at runtime | Typical contents |
|-------|--------|-------|--------------------|------------------|
| 1 | Package defaults (in `core/` and each `packages/*`) | Global | No | Fallback pool sizes, sensible timeouts, default feature flags |
| 2 | `config/*.php` files | Global | On reload | Structural wiring, driver maps, non-secret defaults |
| 3 | `.env` → process environment | Global | On restart/reload | Secrets, endpoints, hostnames, env name |
| 4 | MongoDB `settings` collection | Per workspace + website | Yes (live) | Site name, locale, SEO, mail-from, toggles, AI opt-in |

> **Warning**
> Layer 4 can only override keys that are **explicitly declared tenant-overridable** in `config/tenant.php`. Infrastructure secrets (Mongo URI, Redis URL, `APP_KEY`) are *never* tenant-overridable — a tenant cannot repoint the database. Attempting to store such a key in `settings` is rejected by the JSON-Schema validator on that collection.

### 2.1 How a value is read

Application code reads config through a single typed accessor. It resolves layers 1–3 (the process-global merge) and is pure/synchronous:

```php
use function Goco\config;

$env      = config('app.env', 'production');        // string, with default
$poolSize = config()->int('mongodb.pool_size', 16); // typed helpers
$driver   = config()->enum('storage.driver', ['local','minio','s3']);
```

For **tenant-scoped** product settings (layer 4), read through the settings service, which resolves website → workspace → global fallback:

```php
use Goco\Settings\Settings;

// Resolves for the current request's website, falling back to workspace, then config().
$siteName = Settings::get('site.name');
$locale   = Settings::get('site.locale', 'en_US');

// Explicit scope:
$fromAddr = Settings::forWebsite($websiteId)->get('mail.from_address');
```

---

## 3. The `config/*.php` Files

Each concern gets one file under `config/` at the project root. Every file returns an array whose values pull from the environment via the `env()` helper — this is the bridge between layer 2 and layer 3.

```
config/
├── app.php          # name, url, env, key, timezone, locale, debug
├── database.php     # MongoDB uri, db name, pool, read/write concern
├── redis.php        # url, pools, prefixes for cache/queue/session/lock
├── cache.php        # store, ttl defaults, tags
├── queue.php        # connection, queues, retry/backoff, worker concurrency
├── session.php      # driver (redis), lifetime, cookie, same-site
├── storage.php      # default driver + local/minio/s3 disks
├── search.php       # provider + mongo/meilisearch/opensearch settings
├── mail.php         # transport (mailpit/smtp), from, dsn
├── traefik.php      # domains, tls resolver, entrypoints (used by deploy tooling)
├── ai.php           # providers, models, credentials, budgets
├── security.php     # hashing, csrf, cors, rate limits, headers
└── tenant.php       # which setting keys are tenant-overridable
```

A representative file:

```php
<?php
// config/app.php
return [
    'name'     => env('APP_NAME', 'GOCO CMS'),
    'url'      => env('APP_URL', 'http://localhost:8080'),
    'env'      => env('APP_ENV', 'production'),   // production|staging|development|testing
    'debug'    => env('APP_DEBUG', false),        // env() casts "true"/"false" to bool
    'key'      => env('APP_KEY'),                 // base64:... — REQUIRED, no default
    'timezone' => env('APP_TIMEZONE', 'UTC'),
    'locale'   => env('APP_LOCALE', 'en_US'),
    'mode'     => env('APP_MODE', 'coroutine'),   // maps to App::MODE_COROUTINE
];
```

> **Tip**
> The `env()` helper casts common strings: `"true"/"false"` → bool, `"null"` → null, `"(empty)"` → `''`, and numeric strings stay strings unless you use `env_int()` / `env_bool()`. Never call `getenv()` in application code — it bypasses casting and, under OpenSwoole, is not coroutine-isolated.

---

## 4. Key Configuration Groups

The following examples show the `.env` keys (layer 3) that feed the `config/*.php` files. A complete `.env.example` ships in the repo root; copy it to `.env` and fill in secrets.

### 4.1 APP

```env
APP_NAME="Acme Sites"
APP_URL=https://cms.acme.test
APP_ENV=production
APP_DEBUG=false
APP_KEY=base64:Y2hhbmdlLW1lLXRvLTMyLWJ5dGVzLW9mLXJhbmRvbSE=
APP_TIMEZONE=UTC
APP_LOCALE=en_US
APP_MODE=coroutine          # coroutine | legacy_cgi | coroutine_legacy | mixed
```

`APP_KEY` is a 32-byte key, base64-encoded, used for cookie/session signing and at-rest field encryption. Generate it with the CLI:

```bash
goco key:generate            # writes APP_KEY into .env
```

> **Warning**
> Rotating `APP_KEY` invalidates all signed cookies and any encrypted fields written with the old key. Use `goco key:rotate` (dual-key window) rather than editing `.env` by hand in production.

### 4.2 MongoDB — primary database

```env
MONGODB_URI=mongodb://gococms:secret@mongodb:27017/?authSource=admin&replicaSet=rs0
MONGODB_DB=gococms
MONGODB_POOL_SIZE=16
MONGODB_READ_CONCERN=majority
MONGODB_WRITE_CONCERN=majority
MONGODB_MAX_STALENESS=90
```

```php
<?php
// config/database.php
return [
    'uri'  => env('MONGODB_URI', 'mongodb://mongodb:27017'),
    'db'   => env('MONGODB_DB', 'gococms'),
    'pool' => [
        'size'         => env_int('MONGODB_POOL_SIZE', 16), // one pool per worker
        'max_idle_ms'  => env_int('MONGODB_POOL_MAX_IDLE', 60000),
    ],
    'read_concern'  => env('MONGODB_READ_CONCERN', 'majority'),
    'write_concern' => env('MONGODB_WRITE_CONCERN', 'majority'),
];
```

> **Note**
> Multi-document transactions and change streams require a replica set (`replicaSet=rs0`), even for a single node. The bundled `docker-compose.yml` initializes `mongodb` as a one-node replica set for this reason. See [MongoDB Data Layer](../architecture/database-mongodb.md).

### 4.3 Redis — cache, queue, sessions, locks, realtime

A single Redis is used for many roles, separated by key prefix (and optionally logical DB index).

```env
REDIS_URL=redis://:secret@redis:6379/0
REDIS_POOL_SIZE=32
REDIS_PREFIX=goco:
CACHE_STORE=redis
CACHE_TTL=3600
SESSION_DRIVER=redis
SESSION_LIFETIME=1209600      # seconds (14 days)
QUEUE_CONNECTION=redis
QUEUE_DEFAULT=default
```

```php
<?php
// config/redis.php
return [
    'url'    => env('REDIS_URL', 'redis://redis:6379'),
    'prefix' => env('REDIS_PREFIX', 'goco:'),
    'pool'   => ['size' => env_int('REDIS_POOL_SIZE', 32)],
    'channels' => [
        'cache'   => env('REDIS_PREFIX', 'goco:') . 'cache:',
        'session' => env('REDIS_PREFIX', 'goco:') . 'sess:',
        'queue'   => env('REDIS_PREFIX', 'goco:') . 'queue:',
        'lock'    => env('REDIS_PREFIX', 'goco:') . 'lock:',
        'rate'    => env('REDIS_PREFIX', 'goco:') . 'rl:',
    ],
];
```

Redis also backs `\ZealPHP\Store` when you opt into `Store::defaultBackend(Store::BACKEND_REDIS)` for cross-worker/cross-node shared state. See [Caching, Queue & Realtime](../architecture/caching-and-queue.md).

### 4.4 Storage — object storage driver

The storage layer is a driver interface with three implementations: `local`, `minio`, `s3`. Select the default disk and configure each disk independently.

```env
STORAGE_DRIVER=minio          # local | minio | s3

# MinIO (S3-compatible, self-hosted)
MINIO_ENDPOINT=http://minio:9000
MINIO_KEY=minioadmin
MINIO_SECRET=minioadmin
MINIO_BUCKET=goco-media
MINIO_REGION=us-east-1
MINIO_USE_PATH_STYLE=true

# Amazon S3
S3_KEY=AKIA...
S3_SECRET=...
S3_BUCKET=acme-goco-media
S3_REGION=eu-west-1
```

```php
<?php
// config/storage.php
return [
    'default' => env('STORAGE_DRIVER', 'local'),
    'disks' => [
        'local' => ['root' => env('STORAGE_LOCAL_ROOT', 'storage/media')],
        'minio' => [
            'endpoint'         => env('MINIO_ENDPOINT'),
            'key'              => env('MINIO_KEY'),
            'secret'           => env('MINIO_SECRET'),
            'bucket'           => env('MINIO_BUCKET', 'goco-media'),
            'region'           => env('MINIO_REGION', 'us-east-1'),
            'use_path_style'   => env_bool('MINIO_USE_PATH_STYLE', true),
        ],
        's3' => [
            'key'    => env('S3_KEY'),
            'secret' => env('S3_SECRET'),
            'bucket' => env('S3_BUCKET'),
            'region' => env('S3_REGION', 'us-east-1'),
        ],
    ],
];
```

See [Storage & Media](../architecture/storage.md) for signed URLs, image derivatives, and per-workspace bucket prefixes.

### 4.5 Search — swappable provider

```env
SEARCH_PROVIDER=meilisearch    # mongo | meilisearch | opensearch

MEILISEARCH_HOST=http://meilisearch:7700
MEILISEARCH_KEY=masterKeyChangeMe

OPENSEARCH_HOSTS=https://opensearch:9200
OPENSEARCH_USER=admin
OPENSEARCH_PASS=admin
```

```php
<?php
// config/search.php
return [
    'provider' => env('SEARCH_PROVIDER', 'mongo'),
    'providers' => [
        'mongo'       => ['index' => 'default', 'atlas' => env_bool('SEARCH_ATLAS', false)],
        'meilisearch' => ['host' => env('MEILISEARCH_HOST'), 'key' => env('MEILISEARCH_KEY')],
        'opensearch'  => ['hosts' => env('OPENSEARCH_HOSTS'), 'auth' => [env('OPENSEARCH_USER'), env('OPENSEARCH_PASS')]],
    ],
];
```

With `SEARCH_PROVIDER=mongo`, GOCO uses MongoDB text indexes (or Atlas Search) and requires no extra container — ideal for small deployments. See [Search](../architecture/search.md).

### 4.6 Mail — Mailpit in dev, SMTP in prod

```env
# Development (bundled Mailpit — captures all mail, no real delivery)
MAIL_TRANSPORT=smtp
MAIL_DSN=smtp://mailpit:1025
MAIL_FROM_ADDRESS=no-reply@acme.test
MAIL_FROM_NAME="Acme Sites"
```

```env
# Production (real SMTP)
MAIL_TRANSPORT=smtp
MAIL_DSN=smtps://apikey:SG.xxxx@smtp.sendgrid.net:465
MAIL_FROM_ADDRESS=no-reply@acme.com
MAIL_FROM_NAME="Acme"
```

`mail.from_address` and `mail.from_name` are tenant-overridable (layer 4), so each website can send from its own identity while sharing one transport. Preview all captured mail at the Mailpit UI (`http://localhost:8025` by default).

### 4.7 Traefik — domains & TLS

Traefik is the reverse proxy and handles automatic HTTPS. Its runtime config lives in Docker labels (see [Docker Architecture](../deployment/docker.md) and [Traefik](../deployment/traefik.md)), but GOCO reads the same values so it can generate correct absolute URLs and validate incoming `Host` headers.

```env
TRAEFIK_DOMAINS=cms.acme.test,*.sites.acme.test
TRAEFIK_ENTRYPOINT=websecure
TRAEFIK_TLS_RESOLVER=letsencrypt
ACME_EMAIL=ops@acme.com
TRAEFIK_HTTP3=true
```

```php
<?php
// config/traefik.php
return [
    'domains'      => explode(',', env('TRAEFIK_DOMAINS', 'localhost')),
    'entrypoint'   => env('TRAEFIK_ENTRYPOINT', 'websecure'),
    'tls_resolver' => env('TRAEFIK_TLS_RESOLVER', 'letsencrypt'),
    'acme_email'   => env('ACME_EMAIL'),
    'http3'        => env_bool('TRAEFIK_HTTP3', true),
];
```

> **Note**
> Per-tenant custom domains are stored in the `domains` collection and rendered into Traefik routers dynamically; they do **not** require redeploying. `TRAEFIK_DOMAINS` covers only the base/wildcard hosts owned by the platform.

### 4.8 AI Platform credentials

```env
AI_ENABLED=true
AI_DEFAULT_PROVIDER=anthropic
AI_DEFAULT_MODEL=claude-opus-4-5
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
AI_MONTHLY_BUDGET_USD=250        # soft cap; enforced per workspace
AI_REQUEST_TIMEOUT=60
```

```php
<?php
// config/ai.php
return [
    'enabled'  => env_bool('AI_ENABLED', false),
    'default'  => ['provider' => env('AI_DEFAULT_PROVIDER', 'anthropic'), 'model' => env('AI_DEFAULT_MODEL')],
    'providers' => [
        'anthropic' => ['key' => env('ANTHROPIC_API_KEY')],
        'openai'    => ['key' => env('OPENAI_API_KEY')],
    ],
    'budget' => ['monthly_usd' => env_int('AI_MONTHLY_BUDGET_USD', 0)],
    'timeout' => env_int('AI_REQUEST_TIMEOUT', 60),
];
```

`ai.enabled` is a tenant-overridable toggle (requires the `ai.manage` capability), and budget is enforced per workspace. See the [AI Platform](../core/ai-platform.md).

### 4.9 Session, Cache & Queue

All three ride on Redis by default. Sessions use ZealPHP's per-coroutine `$_SESSION` isolation with a Redis-backed handler.

```env
SESSION_DRIVER=redis
SESSION_LIFETIME=1209600
SESSION_COOKIE=goco_session
SESSION_SAME_SITE=lax
SESSION_SECURE=true            # force Secure cookie flag (behind Traefik TLS)

CACHE_STORE=redis
CACHE_TTL=3600

QUEUE_CONNECTION=redis
QUEUE_DEFAULT=default
QUEUE_RETRY_AFTER=90
QUEUE_MAX_ATTEMPTS=3
QUEUE_WORKER_CONCURRENCY=8
```

Queue workers are driven by ZealPHP coroutines/timers (`App::onWorkerStart` → `App::tick`), not a separate PHP process per job. See [Caching, Queue & Realtime](../architecture/caching-and-queue.md).

### 4.10 Security

```env
HASH_DRIVER=argon2id
ARGON_MEMORY=65536
ARGON_TIME=4
ARGON_THREADS=2

CSRF_ENABLED=true
CORS_ALLOWED_ORIGINS=https://cms.acme.test
RATE_LIMIT_GLOBAL=600          # requests/min/IP at the edge
RATE_LIMIT_LOGIN=10            # login attempts/min/IP
SECURITY_HEADERS=true          # HSTS, X-Content-Type-Options, frame-ancestors
```

CSRF is enforced by the ZealPHP `Csrf` middleware; rate limiting by the `RateLimit` middleware (Redis-backed); security headers by `security.php` + Traefik middleware. Password hashing is **Argon2id**. See the [Security Model](../security/security-model.md) and [Authentication](../core/authentication.md).

---

## 5. Precedence Rules

When the same logical key exists in multiple layers, resolution is deterministic:

```mermaid
flowchart TD
    Q{Key requested} --> T{Tenant-overridable<br/>& tenant scope active?}
    T -- "yes, found in settings" --> S[Use settings value]
    T -- "no / not found" --> E{Present in<br/>.env / process env?}
    E -- yes --> V[Use env value]
    E -- no --> C{Present in<br/>config/*.php?}
    C -- yes --> W[Use config value]
    C -- no --> D[Use package default]
```

Concrete examples:

| Key | Layer 2 (`config`) | Layer 3 (`.env`) | Layer 4 (`settings`) | Effective |
|-----|--------------------|-------------------|-----------------------|-----------|
| `app.name` | `"GOCO CMS"` | `APP_NAME="Acme"` | — | `"Acme"` |
| `site.name` (website A) | — | — | `"Acme Blog"` | `"Acme Blog"` |
| `mongodb.uri` | `mongodb://mongodb:27017` | `MONGODB_URI=...rs0` | *(rejected)* | `.env` value |
| `mail.from_address` | `no-reply@localhost` | `no-reply@acme.test` | `hello@blog.acme.test` (website B) | tenant value for B, `.env` value elsewhere |

> **Warning**
> A tenant setting only wins when a tenant scope is resolved for the current request. Background jobs and CLI commands run **without** a website context unless you explicitly bind one with `Settings::forWebsite($id)`. Never assume a tenant override applies inside a queue worker.

---

## 6. Environment Presets

`APP_ENV` selects behavior bundles. GOCO ships four canonical environments; the value flips several defaults so you don't have to set each flag by hand.

| `APP_ENV` | `APP_DEBUG` default | Mail | Cache | Error output | HTTPS enforce |
|-----------|---------------------|------|-------|--------------|---------------|
| `development` | `true` | Mailpit | array/no-op ok | full stack traces | off |
| `testing` | `true` | in-memory (`null` transport) | array | assertions verbose | off |
| `staging` | `false` | real SMTP | Redis | logged, generic page | on |
| `production` | `false` | real SMTP | Redis | logged, generic page | on |

You still override individual keys explicitly; the preset only changes *defaults*, and an explicit `.env` value always wins.

---

## 7. Secrets Handling

- **Never commit `.env`.** It is git-ignored by default. Commit only `.env.example` with **empty or dummy** values and inline comments.
- **Generate, don't invent, the app key.** Use `goco key:generate`; rotate with `goco key:rotate`.
- **Prefer real secret managers in production.** In Docker/Kubernetes, inject secrets as environment variables via Docker secrets, a `.env` mounted read-only, or your platform's secret store. GOCO reads whatever is in the process environment — `.env` is just a convenient local loader.
- **Redact in logs.** The config layer marks keys matching `*_KEY`, `*_SECRET`, `*_PASS`, `*_TOKEN`, `*_DSN`, and `MONGODB_URI` as sensitive; they are masked in `goco config:show`, health output, and structured logs.
- **Encrypt sensitive fields at rest.** Tenant secrets stored in `settings` (e.g. a per-workspace SMTP password) are encrypted with `APP_KEY` before insertion and decrypted on read — they are never stored in plaintext in MongoDB.

Inspect the effective, merged configuration (secrets masked) without leaking anything:

```bash
goco config:show                 # all groups, secrets masked
goco config:show mongodb         # one group
goco config:show --raw app.key   # masked as "base64:****" unless --unsafe
goco config:validate             # type-check + required-key check, exits non-zero on error
```

---

## 8. Reloading Config in a Long-Running Worker

Because OpenSwoole workers are persistent, editing `.env` or `config/*.php` does **not** take effect until the config is reloaded into memory. There are three mechanisms, in increasing scope:

### 8.1 Live tenant settings (no reload)

Layer 4 (`settings` collection) is read through the settings service, which caches per-website in Redis with a short TTL and a pub/sub invalidation channel. Changing a tenant setting in the admin UI publishes an invalidation, and the next request on any worker sees the new value — no restart. This is the intended path for editors changing site name, locale, or toggles.

```php
// Under the hood: settings service subscribes on worker start.
App::onWorkerStart(function ($server, $wid) {
    App::subscribe('goco:settings:invalidate', function ($msg) {
        Settings::flushLocal($msg['website_id']);
    });
});
```

### 8.2 Graceful reload of layers 2–3 (SIGUSR1 / CLI)

To pick up edited `config/*.php` or a changed `.env`, reload workers **gracefully** — OpenSwoole finishes in-flight requests, then restarts workers, which re-parse config at start:

```bash
goco config:clear      # drop cached compiled config
php app.php restart     # graceful worker reload (drains connections)
```

In containers, the same is achieved with a rolling restart of the `gococms` service. No dropped requests when behind Traefik with healthchecks.

### 8.3 Full restart (changing pool sizes, ports, mode)

Structural settings — `APP_MODE`, worker/pool sizing, listen port, middleware stack — are read only at `App::init()` and require a **full process restart**:

```bash
php app.php stop && php app.php start -d
# or, in Docker:
docker compose up -d --force-recreate gococms
```

> **Tip**
> Treat config changes like deploys: change `.env` → `goco config:validate` → rolling restart. Never hot-edit config on a single node in a cluster; push the change through your deploy pipeline so every replica converges.

---

## 9. A Minimal Working `.env`

The smallest `.env` that boots a single-node development stack from the bundled `docker-compose.yml`:

```env
APP_NAME="GOCO Dev"
APP_URL=http://localhost:8080
APP_ENV=development
APP_KEY=base64:REPLACE_WITH_goco_key_generate
APP_MODE=coroutine

MONGODB_URI=mongodb://mongodb:27017/?replicaSet=rs0
MONGODB_DB=gococms

REDIS_URL=redis://redis:6379/0

STORAGE_DRIVER=local
SEARCH_PROVIDER=mongo

MAIL_TRANSPORT=smtp
MAIL_DSN=smtp://mailpit:1025
MAIL_FROM_ADDRESS=dev@localhost
```

Run `goco key:generate` first, then `docker compose up -d`. Continue with [Quick Start](quick-start.md).

---

## Related

- [Installation](installation.md) — bring up the Docker stack these values configure
- [Quick Start](quick-start.md) — first workspace, website, and page
- [Project Structure](project-structure.md) — where `config/`, `.env`, and `app.php` live
- [Configuration Reference](../reference/configuration-reference.md) — exhaustive key-by-key table
- [Multi-Tenancy](../architecture/multi-tenancy.md) — per-workspace / per-website overrides
- [Caching, Queue & Realtime (Redis)](../architecture/caching-and-queue.md)
- [Storage & Media](../architecture/storage.md) · [Search](../architecture/search.md)
- [Docker Architecture](../deployment/docker.md) · [Traefik Reverse Proxy](../deployment/traefik.md)
- [Security Model](../security/security-model.md) · [Authentication](../core/authentication.md)
- [CLI Reference](../reference/cli-reference.md) — `goco config:*` and `key:*` commands
- [Documentation Home](../README.md)
