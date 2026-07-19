# Security Model

> The complete security architecture, threat model, and coordinated-disclosure policy for GOCO CMS — this document is the canonical `SECURITY.md` for the project.

GOCO CMS — "The Open Source Website Operating System" — is a multi-tenant platform that runs untrusted third-party plugins, hosts many independent websites in one deployment, and exposes public HTTP, WebSocket, and API surfaces. Security is therefore not a feature bolted on at the edge; it is a property of every layer, from the Traefik ingress down to the MongoDB document validators.

This page describes **how GOCO defends itself** (architecture, controls, and hardening) and **how to report a vulnerability responsibly** (disclosure policy, supported versions, and the coordinated release process). It is written for operators, plugin developers, and security researchers alike.

> **Note**
> Security is layered with related subsystems. This document is the umbrella; deep mechanics live in [Authentication](../core/authentication.md), the [Permission System](../architecture/permission-system.md), [Multi-Tenancy](../architecture/multi-tenancy.md), and the [Plugin Engine](../core/plugin-engine.md). Cross-links appear throughout and in [Related](#related).

---

## Security Design Principles

GOCO's security posture follows a small set of non-negotiable principles. Every control below traces back to one of these.

| Principle | What it means in GOCO |
|-----------|-----------------------|
| **Secure by default** | Fresh installs ship with CSRF on, security headers on, HTTPS enforced by Traefik, soft deletes, audit logging, and least-privilege roles. You opt *out*, never *in*. |
| **Defense in depth** | No single control is trusted. Tenant isolation is enforced at the repository *and* the query *and* the index level; XSS is stopped at output encoding *and* CSP. |
| **Least privilege** | Roles map to `resource.action` capabilities scoped to `(workspace, website)`. Plugins declare the exact capabilities they need; nothing is implicit. |
| **Fail closed** | An authorization check that cannot be evaluated denies. A missing tenant scope raises, it does not "match all". A signature that cannot be verified rejects the package. |
| **Complete mediation** | Every request passes the middleware pipeline; every data access goes through `Goco\Database` repositories that inject tenant scope — there is no "back door" raw query path in application code. |
| **Auditability** | Security-relevant actions land in the immutable `audit_logs` collection with actor, tenant, IP, and before/after context. |
| **Minimal attack surface** | ZealPHP on OpenSwoole means one long-lived PHP process, no per-request PHP-FPM fork, no `.php` files served from web roots by a generic interpreter, and file-based REST only under explicit `api/` roots. |

---

## Threat Model & Trust Boundaries

GOCO is modeled as five nested trust zones. Data crossing a boundary is untrusted until validated, authenticated, and authorized for the destination zone.

```mermaid
flowchart TB
  subgraph Internet["Untrusted Internet"]
    U[Visitors / API clients / Attackers]
  end
  subgraph Edge["Edge Zone — Traefik"]
    T[Traefik: TLS termination, HTTP/3,\nrate limit, security headers, IP allow/deny]
  end
  subgraph App["Application Zone — ZealPHP / OpenSwoole"]
    MW[Middleware pipeline:\nCors, Csrf, RateLimit, BasicAuth, IpAccess]
    R[Router + Auth + PolicyEngine]
    PL[Plugin sandbox\ncapability-gated]
  end
  subgraph Data["Data Zone — private network"]
    M[(MongoDB)]
    RD[(Redis)]
    OS[(MinIO / S3)]
    SE[(Meilisearch)]
  end
  U -->|HTTPS/443, HTTP3| T
  T -->|internal, mTLS optional| MW
  MW --> R
  R --> PL
  R --> Data
  PL -.capability API only.-> R
```

### Trust boundaries and what each enforces

| Boundary | Separates | Primary controls |
|----------|-----------|------------------|
| **Edge** | Internet ↔ app | Traefik: forced HTTPS (Let's Encrypt), HTTP/3, TLS 1.2+, rate limiting, security headers, IP allow/deny, request size limits. Only Traefik is internet-facing. |
| **Application** | Untrusted request ↔ trusted core | ZealPHP middleware (CSRF, CORS, auth), session/JWT verification, `PolicyEngine` authorization, input validation. |
| **Data** | App ↔ datastores | Private Docker network; MongoDB/Redis/MinIO never published to the host's public interface; auth + TLS on every datastore connection. |
| **Tenant** | Website A ↔ Website B | `workspace_id` + `website_id` scoping on every tenant document and query; optional database-per-workspace (enterprise). See [Multi-Tenancy](../architecture/multi-tenancy.md). |
| **Plugin** | Untrusted plugin code ↔ core | Capability gating, hook-only integration surface, no direct datastore handles, signed packages. See [Plugin sandboxing](#plugin-security-sandboxing-supply-chain). |

### Assets, actors, and STRIDE mapping

**Assets:** tenant content (pages/posts/media), user credentials and sessions, PII in `users`/`form_submissions`, secrets (JWT keys, datastore creds, OAuth secrets), the `audit_logs` trail, and platform availability.

**Actors:** anonymous visitor, authenticated user (13 roles from `owner` to `guest`), plugin author, workspace operator, and external attacker.

| STRIDE threat | Example against GOCO | Primary mitigation |
|---------------|----------------------|--------------------|
| **Spoofing** | Session/JWT forgery, phished login | Argon2id passwords, signed JWT, 2FA/TOTP, WebAuthn passkeys, Redis-backed server sessions |
| **Tampering** | Mutating another tenant's doc, forged form POST | Tenant-scoped repositories, CSRF middleware, JSON-Schema validators, optimistic `version` field |
| **Repudiation** | "I didn't publish that" | Immutable `audit_logs` with actor + IP + diff |
| **Information disclosure** | Cross-tenant read, SSRF to metadata endpoint, verbose errors | Scoped queries, SSRF egress allowlist, sanitized error responses, TLS in transit + encryption at rest |
| **Denial of service** | Request floods, coroutine exhaustion, huge uploads | Traefik + `RateLimit`/`ConcurrencyLimit` middleware, upload size caps, Redis rate limiters |
| **Elevation of privilege** | Plugin grabbing `users.manage`, IDOR to admin | Capability gating, `fail-closed` PolicyEngine, no ambient authority for plugins |

---

## Authentication & Authorization

Authentication answers *who are you*; authorization answers *what may you do here*. GOCO separates them cleanly and both are covered in dedicated documents — this section summarizes the guarantees.

### Authentication (identity)  `stable`

- **Passwords** are hashed with **Argon2id** (memory-hard, tuned cost parameters); plaintext is never stored or logged.
- **Sessions** live server-side in **Redis** with per-coroutine isolation via ext-zealphp's `$_SESSION` override. Cookies are `HttpOnly`, `Secure`, `SameSite=Lax` (Strict for admin), rotated on privilege change, and idle/absolute-expiry bounded.
- **API auth** uses short-lived **JWT** (asymmetric signing) plus refresh tokens, and **OAuth2** for third-party clients.
- **Second factor**: **TOTP 2FA** and **WebAuthn passkeys** (phishing-resistant, preferred for admin roles).
- **CSRF** is enforced by the ZealPHP **`Csrf` middleware** on all state-changing browser requests (see below).

Full flows, token lifetimes, and configuration: **[Authentication](../core/authentication.md)**.

### Authorization (permission)  `stable`

GOCO uses **RBAC with optional ABAC**. Thirteen hierarchical roles (`owner`, `super-admin`, `website-admin`, `developer`, `designer`, `editor`, `author`, `seo-manager`, `marketing`, `moderator`, `support`, `viewer`, `guest`) map to `resource.action` capabilities (`pages.publish`, `plugins.manage`, `users.manage`, …). Every check is **scoped to a `(workspace, website)` pair** and evaluated by the `PolicyEngine`, which **fails closed**.

```php
use Goco\SDK\Hook;

// Authorization is enforced in the request path; plugins can only gate on
// capabilities they were granted at registration time.
if (!$ctx->user()->can('pages.publish', $ctx->website())) {
    return $response->status(403)->json(['error' => 'forbidden']);
}
```

ABAC `PolicyEngine` rules add attribute conditions (ownership, time windows, content status) on top of RBAC for fine-grained cases. Design, role/capability matrix, and policy syntax: **[Permission System](../architecture/permission-system.md)**.

---

## Application-Layer Defenses

The following controls run inside the ZealPHP application zone. They are configured on by default and reinforced at the edge.

### CSRF — ZealPHP `Csrf` middleware  `stable`

All cookie-authenticated, state-changing requests (`POST`/`PUT`/`PATCH`/`DELETE`) require a valid anti-CSRF token. GOCO registers the built-in middleware globally and issues a per-session token exposed to templates and htmx.

```php
require 'vendor/autoload.php';
use ZealPHP\App;

App::superglobals(false);
$app = App::init('0.0.0.0', 8080);

// Double-submit + synchronizer token; rejects mismatches with 403.
$app->addMiddleware(new \ZealPHP\Middleware\Csrf());

$app->run();
```

- Token is bound to the Redis session, rotated on login and on privilege escalation.
- **Pure-JWT API routes are exempt** by design (no ambient cookie authority to abuse) — they rely on the `Authorization` header instead.
- htmx regions rendered via `App::fragment()` automatically include the token header.

### XSS — output encoding in the rendering pipeline  `stable`

The [Rendering Pipeline](../architecture/rendering-pipeline.md) treats all dynamic values as untrusted and **context-aware encodes at output**, not at input:

- HTML body → HTML-entity encode; HTML attributes → attribute encode; URLs → URL encode + scheme allowlist (`http`, `https`, `mailto`); JS/JSON contexts → JSON-encode.
- Widget output passing through the `widget.output` filter and template variables are escaped by default; raw output requires an **explicit, audited** opt-in (`raw()`), never the default.
- User-authored rich content (blog posts, comments) is sanitized through an allowlist HTML sanitizer before storage *and* re-encoded on render (defense in depth).
- A **Content-Security-Policy** (see [security headers](#clickjacking-security-headers-traefik)) provides a second line against injected script.

### SSTI — Server-Side Template Injection  `stable`

The [Template Engine](../core/template-engine.md) never compiles user-supplied strings as templates. User input is **data passed to templates**, not template source.

- Template *sources* come only from trusted theme/plugin packages on disk; `App::render()`/`renderToString()` receive a fixed template path plus a data array.
- The Page Builder stores structured JSON (Section → Container → Row → Column → Widget), not template code, so a designer cannot inject expression syntax.
- Plugin-provided templates are loaded from the plugin's signed package directory only.

### Injection — MongoDB query building & validation  `stable`

GOCO uses **MongoDB**, so classic SQL injection does not apply — but **operator injection** ($where, `$gt`/`$ne` smuggling, JS execution) does. Defenses:

- All queries are built through the `Goco\Database` **document-mapper + Repository** layer, which binds values as **typed BSON**, never string-concatenated queries.
- User input is **type-cast and schema-validated** before it reaches a filter; request bodies are decoded into typed DTOs, so a `{"$ne": null}` payload arrives as a *string*, not an operator.
- Server-side JavaScript execution (`$where`, `mapReduce` with JS, `$function`) is **disabled** on the MongoDB deployment.
- Every collection carries a **JSON-Schema validator** (`$jsonSchema`) so malformed documents are rejected by the database itself. See [Data Model](../architecture/data-model.md).

```php
use Goco\Database\Repository;

// SAFE: value is bound as a typed string, operators cannot be smuggled.
$page = $pages->findOne([
    'slug'        => (string) $request->query('slug'),
    'website_id'  => $ctx->websiteId(),   // tenant scope injected by the repo
    'deleted_at'  => null,
]);
```

### SSRF — Server-Side Request Forgery  `stable`

GOCO makes outbound requests for webhooks, remote media import, OAuth, the AI Platform, and plugin HTTP clients. The shared HTTP client enforces:

- A **default-deny egress allowlist** — link-local (`169.254.0.0/16`, incl. cloud metadata `169.254.169.254`), loopback, and RFC 1918 ranges are blocked unless explicitly permitted for a self-hosted service.
- **DNS-rebinding protection**: the resolved IP is pinned and re-validated after resolution; redirects are re-checked against the policy.
- Scheme allowlist (`http`/`https` only), timeouts, response-size caps, and no automatic credential forwarding across hosts.
- Media-import and AI-provider destinations use their own narrower allowlists.

### File-upload security  `stable`

Uploads flow through the [Storage & Media](../architecture/storage.md) layer (Local / MinIO / S3 drivers):

- **MIME sniffing** (magic-byte inspection) — the declared `Content-Type` and extension are not trusted; type is derived from content and matched against an allowlist per collection.
- **Size and dimension caps** enforced before persistence; oversized requests are rejected at the middleware and at Traefik.
- Files are stored under **generated keys** (not user-controlled paths) to prevent path traversal; the original filename is metadata only.
- Uploaded assets are served from object storage / a media route, **never executed** — no user-writable directory is on a PHP execution path.
- Optional AV scanning hook (`media.uploaded`) for enterprise deployments; SVGs are sanitized (script/`<foreignObject>` stripped) before serving.

### Clickjacking & security headers (Traefik)  `stable`

Security headers are applied at the **edge** by a Traefik `headers` middleware so they cover every route uniformly, with the application able to tighten CSP per response via the `response.headers` filter.

```yaml
# docker/traefik/dynamic/security-headers.yml
http:
  middlewares:
    goco-security-headers:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        frameDeny: true                     # X-Frame-Options: DENY (anti-clickjacking)
        contentTypeNosniff: true            # X-Content-Type-Options: nosniff
        referrerPolicy: "strict-origin-when-cross-origin"
        browserXssFilter: false             # rely on CSP, not legacy header
        permissionsPolicy: "camera=(), microphone=(), geolocation=()"
        customResponseHeaders:
          Content-Security-Policy: >-
            default-src 'self'; script-src 'self'; object-src 'none';
            base-uri 'self'; frame-ancestors 'none'; upgrade-insecure-requests
```

> **Tip**
> `frame-ancestors 'none'` (CSP) plus `frameDeny` gives belt-and-suspenders clickjacking protection. Admin routes use the strictest CSP; public theme routes may relax `img-src`/`font-src` via the `response.headers` filter.

---

## Plugin Security: Sandboxing, Supply Chain

Plugins are **untrusted code** running in the app zone; they receive the most scrutiny. See [Plugin Engine](../core/plugin-engine.md) and the [Plugin SDK](../sdk/plugin-sdk.md).

### Capability gating

A plugin declares the exact capabilities it needs in its manifest; the operator approves them at install. Nothing is granted implicitly.

```php
use Goco\SDK\Plugin;

Plugin::register('acme-newsletter', [
    'name'         => 'Acme Newsletter',
    'version'      => '1.4.0',
    'capabilities' => ['forms.read', 'notifications.create'], // requested, must be approved
    'routes'       => true,
]);

// Requested caps are enforced by the core; a plugin route that touches
// users.manage without declaring it is denied — fail closed.
Plugin::permissions(['forms.read', 'notifications.create']);
```

### Integration surface = hooks only

- Plugins integrate through **actions/filters** (`Hook::listen`, `Hook::filter`) and the SDK facades — **not** through raw MongoDB/Redis handles. Data access goes through capability-checked, tenant-scoped repositories, so a plugin cannot read another tenant or another website it wasn't scoped to.
- Plugin routes are namespaced (`nsRoute`) and hooks are slug-namespaced, preventing collisions and privilege bleed.
- Resource guards: plugin-initiated work runs under `ConcurrencyLimit` and timeouts so a runaway coroutine cannot starve the worker pool.
- The AI Platform, network egress, and filesystem access are mediated behind capability-gated services (`ai.manage`, SSRF-guarded HTTP client) rather than exposed as free functions.

### Supply chain & package signing

- Marketplace and Composer packages (`gococms/*`) are **cryptographically signed**; the CLI verifies the signature and publisher key before install and refuses unsigned or tampered packages. See [Marketplace](../marketplace/overview.md).
- **Pinned versions + integrity hashes** via `composer.lock`; reproducible installs in CI.
- Post-install, a plugin's declared capabilities are diffed against a previously approved set — a version bump that silently *widens* capabilities requires re-approval.

```bash
# The developer CLI verifies signatures on install.
goco plugin:install acme-newsletter@1.4.0
# -> verifying publisher signature ... OK
# -> capability diff vs approved set ... unchanged
```

---

## Secrets, Encryption & Transport Security

### Secrets management  `stable`

- Secrets (JWT signing keys, datastore credentials, OAuth client secrets, S3/MinIO keys) come from **environment variables / Docker secrets**, never from source control. See [Configuration](../getting-started/configuration.md) and the [Configuration Reference](../reference/configuration-reference.md).
- `.env` files are git-ignored; example files ship with placeholders only.
- Secrets are **redacted from logs** and error output; the logger scrubs known secret keys.
- JWT and encryption keys support **rotation** (overlapping key IDs so in-flight tokens validate during rollover).

```env
# .env — provided via Docker secrets in production, never committed
GOCO_JWT_PRIVATE_KEY_FILE=/run/secrets/jwt_private_key
GOCO_ENCRYPTION_KEY_FILE=/run/secrets/app_encryption_key
MONGODB_URI=mongodb://goco:${MONGO_PASSWORD}@mongodb:27017/goco?tls=true&authSource=admin
REDIS_URL=rediss://:${REDIS_PASSWORD}@redis:6380/0
```

### Encryption in transit  `stable`

- **Traefik terminates TLS** at the edge with automatic Let's Encrypt certificates (wildcard/multi-domain), **HTTP/3**, HSTS with preload, and a TLS 1.2+ floor (1.3 preferred).
- **Datastore connections use TLS**: MongoDB (`tls=true`), Redis (`rediss://`), MinIO/S3 (HTTPS endpoints). The private Docker network is treated as hostile-adjacent, so intra-service traffic is encrypted for sensitive links.

### Encryption at rest  `stable`

- **MongoDB**: encryption-at-rest via the storage engine (self-managed keys or KMS); application-level field encryption for the most sensitive PII fields (e.g. TOTP secrets, OAuth tokens) using the app encryption key.
- **Object storage**: server-side encryption (SSE-S3/SSE-KMS on S3; SSE on MinIO).
- **Backups are encrypted** at rest and in transit — see [Backup & Restore](../deployment/backup-restore.md).

---

## Session & API Security

### Session security  `stable`

- Server-side sessions in Redis (no sensitive state in the cookie); `HttpOnly` + `Secure` + `SameSite`; ID rotation on login/privilege change; idle and absolute timeouts; concurrent-session limits and remote logout ("sign out everywhere").
- Session data is namespaced by tenant so a coroutine handling Website A never sees Website B's session store.

### API security  `stable`

- **JWT**: asymmetric signing, short expiry, `aud`/`iss`/`exp` validation, refresh-token rotation with reuse detection.
- **OAuth2** for third-party apps with scoped grants that map to GOCO capabilities.
- **Rate limiting & concurrency**: ZealPHP `RateLimit` and `ConcurrencyLimit` middleware (Redis-backed counters) plus a Traefik rate-limit middleware at the edge — layered so an attacker cannot bypass one by hitting a different route.
- **CORS**: the `Cors` middleware enforces an explicit origin allowlist per website; credentials-carrying CORS never uses `*`.
- Consistent, minimal error envelopes (no stack traces, no internal identifiers) and per-key request quotas.

```php
$app->addMiddleware(new \ZealPHP\Middleware\Cors());          // origin allowlist
$app->addMiddleware(new \ZealPHP\Middleware\RateLimit());     // Redis-backed
$app->addMiddleware(new \ZealPHP\Middleware\ConcurrencyLimit());
```

Full endpoint reference: [API Reference](../reference/api-reference.md).

---

## Audit Logging  `stable`

Every security-relevant action is written to the **immutable `audit_logs`** collection (append-only; no application update/delete path).

Each entry records: `actor` (`created_by`), `workspace_id`, `website_id`, `action` (hook name, e.g. `user.login`, `plugin.activated`, `content.published`), source IP + user agent, target resource, and a before/after diff for mutations, plus `created_at`.

```javascript
// MongoDB shell — recent privileged actions for a tenant
db.audit_logs.find({
  website_id: ObjectId("..."),
  action: { $in: ["plugin.activated", "users.manage", "settings.manage"] }
}).sort({ created_at: -1 }).limit(50)
```

Indexes on `(website_id, created_at)` and `(actor, created_at)` keep forensic queries fast. Logs feed the analytics and alerting pipelines (see [Architecture Overview](../architecture/overview.md)) and are covered by the [Backup & Restore](../deployment/backup-restore.md) retention policy.

---

## Datastore & Infrastructure Hardening

All datastores sit on a **private Docker network** and are **never published** to the host's public interface. See [Docker Architecture](../deployment/docker.md) and [Deployment Guide](../deployment/deployment-guide.md).

### MongoDB

- Authentication + RBAC enabled; a least-privilege application user scoped to the GOCO database (no `root`).
- TLS on connections; bind to the private network only; `$where`/server-side JS disabled.
- JSON-Schema validators on every collection; soft deletes (`deleted_at`) + `version` for tamper-evidence; documented indexes prevent unindexed-query DoS.
- Encryption at rest; audited backups.

### Redis

- `requirepass` / ACLs; TLS (`rediss://`); private network only; dangerous commands (`FLUSHALL`, `CONFIG`, `KEYS`) renamed/disabled in production.
- Separate logical databases for sessions, cache, queue, locks, and rate-limit counters; sensible `maxmemory` + eviction policy so cache pressure never evicts session state.

### MinIO / S3

- Scoped access keys per bucket with least-privilege IAM policies; no public-read buckets by default (media served through the app/CDN with signed URLs where private).
- Server-side encryption; TLS endpoints; versioning + object-lock for backup buckets.

### Meilisearch / OpenSearch

- Bound to the private network with an API key; only indexes non-sensitive, tenant-scoped searchable fields. Search remains a swappable provider — see [Search](../architecture/search.md).

---

## Multi-Tenant Isolation Guarantees

Isolation is the defining security property of a "Website Operating System". GOCO guarantees it at multiple layers (full detail in [Multi-Tenancy](../architecture/multi-tenancy.md)).

- **Default (shared DB)**: every tenant document carries `workspace_id` + `website_id`. The `Goco\Database` repositories **inject the current tenant scope into every read and write** from the request context — application code cannot issue an unscoped tenant query, and a missing scope raises rather than matching all.
- **Index-enforced**: compound indexes lead with `website_id`, so scoped queries are both correct and fast and a scan across tenants is not a viable path.
- **Session/cache isolation**: Redis keys and sessions are namespaced per tenant.
- **Enterprise (database-per-workspace)**: hard isolation with a dedicated MongoDB database per workspace for customers requiring physical separation.
- **Routing isolation**: per-tenant Traefik routers map domains to the correct `(workspace, website)`; a request for one domain can never resolve another tenant's content.

> **Warning**
> The single most dangerous class of bug in a multi-tenant CMS is a **missing tenant scope** (IDOR / cross-tenant read). Never bypass the repository layer with an ad-hoc query. If you must add a raw pipeline, it MUST receive `website_id` from `Context` and be covered by a cross-tenant isolation test (see [Testing Strategy](../community/testing-strategy.md)).

---

## Dependency Scanning & CI Security

Supply-chain and pipeline security are enforced continuously — see [Contributing](../community/contributing.md) and [Coding Standards](../community/coding-standards.md).

- **Dependency scanning**: `composer audit` and automated advisories on every PR and on a schedule; pinned `composer.lock`; Dependabot-style update PRs.
- **SAST**: static analysis (PHPStan/psalm at max level) and security linters run in CI; findings block merge.
- **Secret scanning**: pre-commit and CI secret scanners reject committed credentials.
- **Container scanning**: base images and the `gococms` image are scanned for known CVEs; minimal, pinned base images; images signed for release.
- **Signed releases**: tags follow Conventional Commits + Semantic Versioning; release artifacts and marketplace packages are signed.
- **Least-privilege CI**: scoped tokens, no long-lived cloud creds in workflows, protected branches, required reviews for security-sensitive paths (`core/`, `packages/auth`, `packages/database`).

---

## Responsible Disclosure Policy

> This is the authoritative disclosure policy for GOCO CMS. It is the content of the repository's `SECURITY.md`.

We deeply appreciate the security research community. If you believe you have found a security vulnerability in GOCO CMS, please report it **privately** so we can fix it before it is disclosed publicly.

### How to report

- **Email:** `security@gococms.org` (encrypt sensitive reports with our published PGP key).
- **GitHub:** open a **private security advisory** via *Security → Report a vulnerability* on the `gococms/core` repository.
- **Please do not** open public issues, discussions, or pull requests for undisclosed vulnerabilities, and do not disclose publicly until a coordinated fix is released.

**Include in your report:** affected version/commit, component (e.g. `packages/auth`, plugin engine, Traefik config), a clear description, reproduction steps or PoC, and the impact/severity you assess.

### What to expect (coordinated disclosure)

| Stage | Target timeline |
|-------|-----------------|
| Acknowledgement of your report | within **72 hours** |
| Triage + severity (CVSS) assignment | within **7 days** |
| Fix developed & validated | severity-dependent (critical: days) |
| Coordinated release + advisory (GHSA/CVE) | on completion, with credit if you wish |
| Public disclosure | after a patched release, typically **≤ 90 days** by mutual agreement |

### Scope

**In scope:** the core (`gococms/core`, `gococms/cli`, `packages/*`), official plugins/themes/widgets, the default Docker/Traefik deployment, and this repository's code.

**Out of scope:** third-party plugins not published by the GOCO project (report to their authors), volumetric DDoS, social engineering of maintainers, issues requiring a physically compromised host or a malicious operator with `owner` capabilities, and findings against outdated/unsupported versions.

### Safe harbor

Good-faith research that respects user privacy, avoids service degradation, does not exfiltrate or destroy data beyond the minimum needed to prove a finding, and gives us reasonable time to remediate will not be pursued or reported by the GOCO project. Do not access, modify, or delete data that is not your own.

### Supported versions

GOCO is **pre-1.0 and under active development**; security fixes land on the **latest minor** on `main`. Once 1.0 ships, the most recent **two minor** releases receive security patches.

| Version | Supported |
|---------|-----------|
| Latest `main` / newest release | ✅ Yes |
| Previous minor | ✅ (post-1.0) |
| Older / pre-release tags | ❌ Upgrade required |

Track advisories in the [Changelog](../changelog.md); plan upgrades with the [Deployment Guide](../deployment/deployment-guide.md).

---

## Operator Security Checklist

Use this before exposing a GOCO deployment to the internet:

- [ ] Traefik terminating TLS with Let's Encrypt; HSTS + security headers applied; HTTP → HTTPS redirect on.
- [ ] MongoDB, Redis, MinIO, Meilisearch bound to the **private** network only — none published to `0.0.0.0`.
- [ ] Datastore auth enabled; TLS on datastore connections; server-side JS disabled in MongoDB.
- [ ] Secrets provided via Docker secrets / env — no credentials in git; `.env` git-ignored.
- [ ] JWT + app encryption keys generated fresh per deployment; rotation plan in place.
- [ ] CSRF, CORS allowlist, RateLimit, ConcurrencyLimit middleware enabled.
- [ ] 2FA/passkeys enforced for `owner`/`super-admin`/`website-admin`.
- [ ] Only signed plugins installed; capability approvals reviewed.
- [ ] Encrypted, tested backups + verified restore (see [Backup & Restore](../deployment/backup-restore.md)).
- [ ] Audit logging on; log shipping + alerting configured.
- [ ] Dependency, container, and secret scanning green in CI.

---

## Related

- [Authentication](../core/authentication.md)
- [Permission System (RBAC + ABAC)](../architecture/permission-system.md)
- [Multi-Tenancy](../architecture/multi-tenancy.md)
- [Rendering Pipeline](../architecture/rendering-pipeline.md)
- [MongoDB Data Layer](../architecture/database-mongodb.md)
- [Data Model (Collections & Indexes)](../architecture/data-model.md)
- [Storage & Media](../architecture/storage.md)
- [Search](../architecture/search.md)
- [Plugin Engine](../core/plugin-engine.md)
- [Plugin SDK](../sdk/plugin-sdk.md)
- [Template Engine](../core/template-engine.md)
- [Marketplace](../marketplace/overview.md)
- [Docker Architecture](../deployment/docker.md)
- [Traefik Reverse Proxy](../deployment/traefik.md)
- [Deployment Guide](../deployment/deployment-guide.md)
- [Backup & Restore](../deployment/backup-restore.md)
- [Configuration](../getting-started/configuration.md)
- [Configuration Reference](../reference/configuration-reference.md)
- [API Reference](../reference/api-reference.md)
- [Testing Strategy](../community/testing-strategy.md)
- [Contributing](../community/contributing.md)
- [Coding Standards](../community/coding-standards.md)
- [Changelog](../changelog.md)
- [Documentation Index](../README.md)
