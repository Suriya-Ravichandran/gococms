# Glossary

> Canonical, alphabetized reference of every GOCO CMS domain and technology term, with one-to-three-sentence definitions and cross-links to the docs that go deeper.

This is a reference, not a tutorial. Terms are grouped A–Z. Where a concept has a dedicated chapter, follow the link. For the site hierarchy at a glance, see [Data Model](architecture/data-model.md) and the [Domain Model](product/domain-model.md).

- [Conventions](#conventions)
- [A](#a) · [B](#b) · [C](#c) · [D](#d) · [E](#e) · [F](#f) · [H](#h) · [I](#i) · [J](#j) · [L](#l) · [M](#m) · [O](#o) · [P](#p) · [R](#r) · [S](#s) · [T](#t) · [U](#u) · [W](#w)
- [Related](#related)

## Conventions

| Notation | Meaning |
| --- | --- |
| `Goco\` | Root PSR-4 namespace of the core. |
| `Goco\SDK\{Widget,Theme,Plugin,Hook}` | Public facades used by extension authors. |
| `resource.action` | Capability string form (e.g. `pages.publish`). |
| `subject.verb[.tense]` | Hook action name form (e.g. `page.rendered`). |
| `subject.noun` | Hook filter name form (e.g. `page.title`). |
| `stable` / `beta` / `experimental` | Stability of the referenced surface, where relevant. |

## A

**ABAC (Attribute-Based Access Control)** — Optional, fine-grained authorization layer that evaluates attributes (owner, workspace, website, resource state, time) via a [Policy](#p) engine, on top of the primary [RBAC](#r) model. See [Permission System](architecture/permission-system.md).

**Action** — A [Hook](#h) side-effect channel named `subject.verb[.tense]` (e.g. `core.boot`, `page.rendered`, `user.login`). Listeners run for effect and return nothing; contrast with a [Filter](#f). Registered with `Hook::listen()` / `on()`, fired with `Hook::dispatch()` / `do()`. See the [Hook SDK](sdk/hook-sdk.md).

**Aggregation Pipeline** — A [MongoDB](#m) multi-stage query used across GOCO for reporting, analytics, and joins (`$match`, `$group`, `$lookup`, `$facet`). See [MongoDB Data Layer](architecture/database-mongodb.md).

**Argon2id** — The password-hashing algorithm used for stored credentials. See [Authentication](core/authentication.md).

**Atlas Search** — A managed [MongoDB](#m) full-text search backend, one of the swappable [Search](#s) [providers](#p) alongside MongoDB text, [Meilisearch](#m), and OpenSearch. See [Search](architecture/search.md).

**Audit Log** — The `audit_logs` collection recording security- and content-sensitive mutations (who, what, when, tenant scope) for compliance and forensics. See [Data Model](architecture/data-model.md).

## B

**Block** — The smallest authored content unit inside a rich-text or structured field (paragraph, image, embed, [Widget](#w) placement). Blocks compose into content; distinct from a widget instance, which is a positioned, configured component in the page tree. See [Page Builder](core/page-builder.md).

**Bootstrap** — The startup sequence that loads `vendor/autoload.php`, calls `App::init()`, wires services, and fires `core.boot`. Entry file is `app.php`. See [ZealPHP Foundation](architecture/zealphp-foundation.md).

## C

**Cache** — Ephemeral key/value storage backed by [Redis](#r), used for rendered fragments, query results, and computed permissions. See [Caching, Queue & Realtime](architecture/caching-and-queue.md).

**Capability** — A `resource.action` permission string (e.g. `pages.create`, `plugins.manage`, `ai.manage`) granted to [Roles](#r) and checked at request time. See [Permission System](architecture/permission-system.md).

**Channel** — An `\OpenSwoole\Coroutine\Channel`: a bounded, CSP-style pipe for passing values between [coroutines](#c) without shared-memory locks. See [ZealPHP Foundation](architecture/zealphp-foundation.md).

**Collection (dynamic)** — A user-defined [Content Type](#c) created in the [Database Builder](core/database-builder.md); its rows are stored in the `collection_entries` collection scoped to the parent `collections` definition. Not to be confused with a physical MongoDB collection. See [Database Builder](core/database-builder.md).

**Collection (MongoDB)** — A physical MongoDB document set (snake_case plural, e.g. `pages`, `posts`, `media`). See [Data Model](architecture/data-model.md).

**Column** — A vertical child of a [Row](#r) in the layout tree, holding [Widgets](#w); the level `Row -> Column -> Widget`. See [Page Builder](core/page-builder.md).

**Container** — A layout node that constrains width/padding and groups [Rows](#r); sits at `Section -> Container -> Row`. See [Rendering Pipeline](architecture/rendering-pipeline.md).

**Content Type** — A schema describing a shape of content — built-in ([Page](#p), [Post](#p)) or dynamic ([Collection](#c)). Defines fields, validation, and [Capabilities](#c). See [Domain Model](product/domain-model.md).

**Coroutine** — A cooperatively-scheduled green thread from [OpenSwoole](#o), created with `go(callable)`; GOCO handlers run in `App::MODE_COROUTINE` for non-blocking concurrency. See [Request Lifecycle](architecture/request-lifecycle.md).

**Counter** — `\ZealPHP\Counter`: an atomic, cross-[worker](#w) integer (`new Counter(0)->increment()->get()`), used for metrics and rate accounting without a round-trip to Redis. See [ZealPHP Foundation](architecture/zealphp-foundation.md).

**CSRF (Cross-Site Request Forgery)** — Attack class mitigated by the ZealPHP `Csrf` [middleware](#m), which validates a per-session token on state-changing requests. See [Security Model](security/security-model.md).

## D

**Data Binding Token** — A templating placeholder (e.g. `{{ page.title }}`, `{{ entry.field }}`) resolved by the [Template Engine](core/template-engine.md) against the render [Context](#c) at output time. See [Template Engine](core/template-engine.md).

**Data Layer** — GOCO's lightweight document-mapper + [Repository](#r) pattern in `packages/database` (`Goco\Database`) over MongoDB — deliberately not a heavy [ORM](#o). See [MongoDB Data Layer](architecture/database-mongodb.md).

**Driver** — A swappable implementation behind an interface, selected by configuration. Object [Storage](#s) drivers are Local, [MinIO](#m), and Amazon S3. See [Storage & Media](architecture/storage.md).

## E

**Entry** — A single record (row) inside a dynamic [Collection](#c), persisted in `collection_entries`. See [Database Builder](core/database-builder.md).

**Event** — A general term for something worth reacting to; in GOCO, events are surfaced through the [Hook](#h) system as [Actions](#a) and can be dispatched asynchronously with `Hook::dispatchAsync()`. See [Event & Hook System](architecture/event-hook-system.md).

## F

**Facade** — A static, discoverable entry point to a subsystem: `Goco\SDK\Widget`, `Theme`, `Plugin`, `Hook`. Facades hide wiring while delegating to services in the [Service Container](#s). See [Widget SDK](sdk/widget-sdk.md).

**Filter** — A [Hook](#h) transformation channel named `subject.noun` (e.g. `page.title`, `menu.items`, `response.headers`). Listeners receive a value and return a (possibly modified) value; contrast with an [Action](#a). Registered with `Hook::filter()`, applied with `Hook::apply()`. See [Hook SDK](sdk/hook-sdk.md).

## H

**Hook** — GOCO's extension mechanism spanning [Actions](#a) (side-effects) and [Filters](#f) (value transforms), exposed via the `Hook` facade with priority ordering and plugin-namespaced names. See [Event & Hook System](architecture/event-hook-system.md).

**htmx** — The hypermedia library that powers GOCO's frontend: server-rendered HTML is progressively enhanced, and interactions swap individual regions rendered by ZealPHP's `App::fragment()` over the wire instead of running a client-side SPA. Paired with [SSE](#s)/[WebSocket](#w) for live updates. See [Rendering Pipeline](architecture/rendering-pipeline.md) and [ZealPHP Foundation](architecture/zealphp-foundation.md).

## I

**Index** — A documented MongoDB index (single-field, compound, text, TTL) that backs a query pattern; tenant-scoped collections lead with `workspace_id, website_id`. See [Data Model](architecture/data-model.md).

## J

**JWT (JSON Web Token)** — Signed, stateless bearer token used for API authentication; interactive sessions instead live in [Redis](#r). See [Authentication](core/authentication.md).

## L

**Layout** — A theme-provided page skeleton defining named [Regions](#r); level `Theme -> Layout -> Section`. Enumerated via `Theme::layouts()` and `Theme::regions()`. See [Theme Engine](core/theme-engine.md).

**Legacy CGI Mode** — `App::MODE_LEGACY_CGI` (and `MODE_COROUTINE_LEGACY`, `MODE_MIXED`), execution modes for running classic PHP scripts under ZealPHP; `MODE_COROUTINE` is the modern default. See [ZealPHP Foundation](architecture/zealphp-foundation.md).

**Let's Encrypt** — The ACME certificate authority [Traefik](#t) uses to auto-provision and renew HTTPS certificates, including wildcard/multi-domain. See [Traefik Reverse Proxy](deployment/traefik.md).

## M

**Mailpit** — The development SMTP server and web mailbox used to capture outbound email locally; a Docker [compose](deployment/docker.md) service. See [Docker Architecture](deployment/docker.md).

**Marketplace** — The distribution channel for GOCO [Widgets](#w), [Themes](#t), and [Plugins](#p), with versioning, capabilities, and install flows. See [Plugin Marketplace](marketplace/overview.md).

**Meilisearch** — A fast, typo-tolerant [Search](#s) [provider](#p), swappable with MongoDB text/Atlas Search and OpenSearch; a Docker compose service. See [Search](architecture/search.md).

**Menu** — An ordered, hierarchical navigation structure stored in `menus` and filterable via the `menu.items` [Filter](#f). See [Data Model](architecture/data-model.md).

**Middleware** — A PSR-15 request/response interceptor registered with `$app->addMiddleware(...)`; built-ins include `Cors`, `ETag`, `Compression`, `Range`, `BasicAuth`, `IpAccess`, `RateLimit`, `ConcurrencyLimit`, `MimeType`, `BodyRewrite`, `HostRouter`, `Csrf`, and `Redirect`. See [Request Lifecycle](architecture/request-lifecycle.md).

**MinIO** — Self-hosted, S3-compatible object storage; one of the [Storage](#s) [drivers](#d) (Local, MinIO, Amazon S3) and a Docker compose service. See [Storage & Media](architecture/storage.md).

**MongoDB** — GOCO's primary database: collections with JSON-Schema validation, aggregation, multi-document transactions, indexes, full-text search, soft deletes, versioning, and audit logs. See [MongoDB Data Layer](architecture/database-mongodb.md).

**Multi-Tenancy** — Isolation of many [Workspaces](#w)/[Websites](#w) in one deployment via `workspace_id` + `website_id` on tenant-scoped docs (default), with optional database-per-workspace (enterprise). See [Multi-Tenancy](architecture/multi-tenancy.md).

## O

**OAuth2** — Delegated-authorization protocol supported for third-party sign-in and API access. See [Authentication](core/authentication.md).

**Observer** — A component that subscribes to model or lifecycle changes (create/update/delete/publish) and reacts, typically wired through [Hooks](#h). See [Event & Hook System](architecture/event-hook-system.md).

**ODM (Object-Document Mapper)** — The mapping layer that hydrates MongoDB documents into typed objects and back; in GOCO it is intentionally lightweight (mapper + [Repository](#r)), not a full [ORM](#o). See [MongoDB Data Layer](architecture/database-mongodb.md).

**OpenSwoole** — The coroutine-based PHP application server (22.1+) that [ZealPHP](#z-and-others) runs on, providing [Workers](#w), [Coroutines](#c), [Channels](#c), timers, and shared-memory [Stores](#s). See [ZealPHP Foundation](architecture/zealphp-foundation.md).

**ORM (Object-Relational Mapper)** — A heavy relational mapping pattern GOCO deliberately avoids in favor of a lightweight document mapper; mentioned only for contrast. See [MongoDB Data Layer](architecture/database-mongodb.md).

## P

**Page** — A standalone content document in `pages` with [Revisions](#r) in `page_revisions`; the primary structured web page built in the [Page Builder](core/page-builder.md). See [Data Model](architecture/data-model.md).

**Passkey / WebAuthn** — Phishing-resistant, public-key credentials (FIDO2/WebAuthn) for passwordless sign-in. See [Authentication](core/authentication.md).

**Plugin** — A packaged extension registered with `Plugin::register()` that adds routes, services, [Hooks](#h), and [Capabilities](#c); lifecycle via `install()`/`boot()`. See [Plugin Engine](core/plugin-engine.md) and the [Plugin SDK](sdk/plugin-sdk.md).

**Policy (ABAC)** — A rule evaluated by the PolicyEngine to allow/deny an action based on attributes, layered over [RBAC](#r) and scoped per (workspace, website). See [Permission System](architecture/permission-system.md).

**Post** — A dated, taxonomy-aware blog document in `posts` with [Revisions](#r) in `post_revisions`. See [Blog Engine](core/blog-engine.md).

**Property Schema** — The typed description of a [Widget](#w)'s configurable props, returned by `Widget::properties()` and used to render editor controls. See [Widget SDK](sdk/widget-sdk.md).

**Provider** — A swappable implementation selected by config behind a capability interface — e.g. [Search](#s) providers (MongoDB text/Atlas, [Meilisearch](#m), OpenSearch). Distinct from a [Service Provider](#s). See [Search](architecture/search.md).

## R

**RBAC (Role-Based Access Control)** — The primary authorization model mapping [Roles](#r) to [Capabilities](#c), scoped per (workspace, website), optionally extended by [ABAC](#a). See [Permission System](architecture/permission-system.md).

**Redis** — The in-memory store powering cache, queue, realtime pub/sub, distributed locks, rate limiting, and sessions. See [Caching, Queue & Realtime](architecture/caching-and-queue.md).

**Region** — A named insertion point within a [Layout](#l) (e.g. `header`, `main`, `sidebar`) enumerated by `Theme::regions()`; where [Sections](#s)/[Widgets](#w) are placed. See [Theme Engine](core/theme-engine.md).

**Repository** — The data-access object in `Goco\Database` that encapsulates queries and persistence for a collection, keeping handlers free of raw driver calls. See [MongoDB Data Layer](architecture/database-mongodb.md).

**Revision** — An immutable prior version of a [Page](#p)/[Post](#p) in `page_revisions`/`post_revisions`, enabling history, diff, and rollback. See [Data Model](architecture/data-model.md).

**Role** — A named grant in the hierarchy `owner, super-admin, website-admin, developer, designer, editor, author, seo-manager, marketing, moderator, support, viewer, guest`, each carrying [Capabilities](#c). See [Permission System](architecture/permission-system.md).

**Route** — A Flask-style URL binding via `$app->route('/hello/{name}', fn($name) => ...)` (also `nsRoute`, `nsPathRoute`, `patternRoute`) plus file-based REST (`api/foo/bar.php` -> `GET /api/foo/bar`). Handlers return `int|array|string|Generator`. See [Routing](core/routing.md).

**Row** — A horizontal layout node holding [Columns](#c); level `Container -> Row -> Column`. See [Page Builder](core/page-builder.md).

## S

**SDK** — The public facade surface (`Goco\SDK\{Widget,Theme,Plugin,Hook}`) plus the [CLI](reference/cli-reference.md) that extension authors build against. See [Widget SDK](sdk/widget-sdk.md).

**Section** — A top-level horizontal band of a [Layout](#l) that groups [Containers](#c); level `Layout -> Section -> Container`. See [Rendering Pipeline](architecture/rendering-pipeline.md).

**Service Container** — The dependency-injection registry that resolves services and their dependencies for handlers, plugins, and facades. See [Service Container & DI](architecture/service-container.md).

**Service Provider** — A registrant that binds services into the [Service Container](#s) and runs boot logic; how packages and [Plugins](#p) wire themselves. See [Service Container & DI](architecture/service-container.md).

**Session** — Per-user server state stored in [Redis](#r); `$_SESSION` is coroutine-isolated by ext-zealphp so concurrent requests never collide. See [Authentication](core/authentication.md).

**Soft Delete** — Non-destructive removal via a `deleted_at` timestamp on every document, preserving history and enabling restore. See [Data Model](architecture/data-model.md).

**SSE (Server-Sent Events)** — One-way server-to-client streaming implemented as a [Generator](#generator) plus `$response->sse()`. See [Caching, Queue & Realtime](architecture/caching-and-queue.md).

**Store** — `\ZealPHP\Store`: shared, cross-[worker](#w) memory over `OpenSwoole\Table` (`Store::make/set/get`) with pub/sub (`Store::publish` + `App::subscribe`) and an optional Redis backend (`Store::defaultBackend(Store::BACKEND_REDIS)`). See [ZealPHP Foundation](architecture/zealphp-foundation.md).

**Storage** — The object-[Driver](#d) abstraction (Local, [MinIO](#m), Amazon S3) for media and files. See [Storage & Media](architecture/storage.md).

**Search** — The [Provider](#p) abstraction (MongoDB text/Atlas Search, [Meilisearch](#m), OpenSearch) for content indexing and query. See [Search](architecture/search.md).

## T

**Task Worker** — An [OpenSwoole](#o) worker that runs blocking or long-lived jobs off the request path (delivered via `task()`), keeping request [Workers](#w) responsive. See [Caching, Queue & Realtime](architecture/caching-and-queue.md).

**Taxonomy** — A classification scheme (e.g. categories, tags) stored in `taxonomies`, whose members are [Terms](#t). See [Data Model](architecture/data-model.md).

**Template** — A `.php` view rendered by the [Template Engine](core/template-engine.md) via `App::render()`/`renderToString()`/`renderStream()`, resolving [Data Binding Tokens](#d). See [Template Engine](core/template-engine.md).

**Tenant** — A single isolated occupant (a [Workspace](#w), and within it a [Website](#w)) in a [multi-tenant](#m) deployment. See [Multi-Tenancy](architecture/multi-tenancy.md).

**Term** — A single member of a [Taxonomy](#t) in `terms`, linked to content via `term_relationships`. See [Data Model](architecture/data-model.md).

**Theme** — A registered package (`Theme::register()`) supplying [Layouts](#l), [Regions](#r), and an [AssetBundle](#a); level `Website -> Theme -> Layout`. See [Theme Engine](core/theme-engine.md) and the [Theme SDK](sdk/theme-sdk.md).

**Traefik** — The Docker-first reverse proxy providing auto-HTTPS via [Let's Encrypt](#l), wildcard/multi-domain routing, HTTP/3, dynamic per-tenant routers, middleware, rate limiting, and security headers. See [Traefik Reverse Proxy](deployment/traefik.md).

**Transaction** — A MongoDB multi-document transaction used to enforce cross-collection invariants (e.g. publishing a page and writing its revision atomically). See [MongoDB Data Layer](architecture/database-mongodb.md).

**2FA / TOTP** — Two-factor authentication using time-based one-time passwords (RFC 6238) as a second factor. See [Authentication](core/authentication.md).

## U

**Upsert** — A write that inserts or updates in one operation; common in [Repository](#r) settings and idempotent jobs. See [MongoDB Data Layer](architecture/database-mongodb.md).

**User** — An authenticated principal in `users`, bound to [Roles](#r) and thus [Capabilities](#c) per (workspace, website). See [Authentication](core/authentication.md).

## W

**Watchtower** — Optional container that watches for and auto-applies updated Docker images; a compose service. See [Docker Architecture](deployment/docker.md).

**WebSocket** — Full-duplex realtime transport bound with `$app->ws('/ws/echo', onMessage: ..., onOpen: ..., onClose: ...)`. See [Caching, Queue & Realtime](architecture/caching-and-queue.md).

**Website** — A single site (domains, theme, pages) inside a [Workspace](#w); level `Workspace -> Website -> Theme`. Tenant-scoped by `website_id`. See [Multi-Tenancy](architecture/multi-tenancy.md).

**Widget** — A reusable, configurable UI component. A widget **definition** is the registered type (`Widget::register($type, $definition)`) with its [Property Schema](#p); a widget **instance** is a placed, prop-configured occurrence in the layout tree (`Column -> Widget`), rendered via `Widget::render()`. See [Widget Engine](core/widget-engine.md) and the [Widget SDK](sdk/widget-sdk.md).

**Worker** — An [OpenSwoole](#o) OS process that serves requests; state shared across workers uses [Store](#s)/[Counter](#c). Lifecycle hooks via `App::onWorkerStart()` with `App::tick()`/`App::after()` timers. See [ZealPHP Foundation](architecture/zealphp-foundation.md).

**Workspace** — The top-level tenant boundary owning [Websites](#w), users, and settings; the root of the hierarchy `Workspace -> Website -> Theme -> Layout -> Section -> Container -> Row -> Column -> Widget`. Scoped by `workspace_id`. See [Multi-Tenancy](architecture/multi-tenancy.md).

## Z and others

**ZealPHP** — The runtime framework (on [OpenSwoole](#o) 22.1+, PHP 8.4+) providing [routing](core/routing.md), [middleware](#m), views, WebSocket/[SSE](#s), coroutines, and shared memory; GOCO's application foundation. Bootstrap: `App::init('0.0.0.0', 8080)->run()`. See [ZealPHP Foundation](architecture/zealphp-foundation.md).

**Generator** — A PHP generator returned by a handler or `App::renderStream()` to stream chunked output (with `co::sleep()` between yields) and to back [SSE](#s). See [Rendering Pipeline](architecture/rendering-pipeline.md).

**Context (`Context` / `Goco\G` / RequestContext)** — Per-request state (current tenant, user, locale) passed to renderers (`Widget::render($type, $props, $ctx)`) and available via `\ZealPHP\G`. See [Request Lifecycle](architecture/request-lifecycle.md).

**AssetBundle** — The set of CSS/JS assets a [Theme](#t) contributes, returned by `Theme::assets()`. See [Theme SDK](sdk/theme-sdk.md).

## Related

- [Documentation Index](README.md)
- [Domain Model (DDD)](product/domain-model.md)
- [Architecture Overview](architecture/overview.md)
- [Data Model (Collections & Indexes)](architecture/data-model.md)
- [Event & Hook System](architecture/event-hook-system.md)
- [Permission System (RBAC + ABAC)](architecture/permission-system.md)
- [Widget SDK](sdk/widget-sdk.md) · [Theme SDK](sdk/theme-sdk.md) · [Plugin SDK](sdk/plugin-sdk.md) · [Hook SDK](sdk/hook-sdk.md)
- [Roadmap](roadmap.md) · [Changelog](changelog.md)
