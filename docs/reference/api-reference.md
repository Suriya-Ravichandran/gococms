# API Reference

> Exhaustive index of every public API in GOCO CMS — the PHP SDK facades and helpers you call from widgets, themes, and plugins (Part A), and the versioned HTTP REST + GraphQL surface served by `apps/api` under `/api/v1` (Part B).

This page is the canonical, at-a-glance directory. Signatures are authoritative; each area links to its deep-dive document for behaviour, examples, and edge cases. Stability tags follow the project convention: `stable`, `beta`, `experimental`, `deprecated`.

- **Part A — PHP API** targets code running *inside* the GOCO runtime (ZealPHP on OpenSwoole, PHP 8.4+). All facades live under `Goco\SDK\` and are safe to call from any coroutine handler.
- **Part B — HTTP API** targets external clients (browsers, mobile apps, integrations) and is served by the `apps/api` application behind Traefik.

> **Note**
> GOCO is pre-1.0 and follows [Semantic Versioning](../glossary.md). Anything tagged `beta` or `experimental` may change without a major-version bump until 1.0. Deprecated members are retained for at least one minor release with a `@deprecated` docblock and a runtime `E_USER_DEPRECATED` notice.

---

## Conventions

| Concept | Rule |
| --- | --- |
| Nullability | `?T` denotes nullable; `mixed` denotes any type. |
| Return `void` | Method mutates state or dispatches; no return value. |
| `Context $ctx` | `Goco\SDK\Context` — the render/request context (workspace, website, user, request). |
| Coroutine safety | Every SDK method is coroutine-safe and non-blocking; blocking I/O is dispatched through OpenSwoole hooks. |
| Facades | Static entry points that resolve concrete services from the [Service Container](../architecture/service-container.md). |
| Errors | SDK methods throw `Goco\Exception\*`; HTTP layer maps these to RFC 7807 problem responses. |

---

# Part A — PHP SDK & Facades

The four public facades — `Widget`, `Theme`, `Plugin`, `Hook` — are the stable contract for extension authors. Everything else in this section is the supporting runtime API (container, router, storage, search, repositories, HTTP helpers, config, cache, queue, auth, CLI) that facades and application code build on.

## Goco\SDK\Widget

`namespace Goco\SDK` · **stable** · deep dive: [Widget SDK](../sdk/widget-sdk.md), [Widget Engine](../core/widget-engine.md)

Register, render, introspect, and preview widgets. A widget is a self-contained, prop-driven UI unit at the leaf of the [website hierarchy](../architecture/data-model.md).

| Method | Signature | Description |
| --- | --- | --- |
| `register` | `register(string $type, array\|callable $definition): void` | Register a widget type from an array manifest or a factory closure. |
| `render` | `render(string $type, array $props, ?Context $ctx = null): string` | Render a widget instance to an HTML string using validated props. |
| `properties` | `properties(string $type): PropertySchema` | Return the typed property schema (fields, defaults, validation) for editor UIs. |
| `preview` | `preview(string $type, array $props = []): string` | Render an isolated preview fragment for the Page Builder, using sample or supplied props. |

```php
use Goco\SDK\Widget;

Widget::register('hero', [
    'label'      => 'Hero Banner',
    'category'   => 'marketing',
    'properties' => [
        'heading'  => ['type' => 'string', 'required' => true],
        'subtext'  => ['type' => 'string'],
        'cta_url'  => ['type' => 'url'],
    ],
    'render' => fn(array $props, Context $ctx): string =>
        Goco\SDK\Theme::assets($ctx->themeSlug())->wrap(
            "<section class='hero'><h1>{$props['heading']}</h1></section>"
        ),
]);
```

## Goco\SDK\Theme

`namespace Goco\SDK` · **stable** · deep dive: [Theme SDK](../sdk/theme-sdk.md), [Theme Engine](../core/theme-engine.md)

Register themes and query their layouts, regions, and asset bundles. A theme sits above layouts in the hierarchy (`Website -> Theme -> Layout`).

| Method | Signature | Description |
| --- | --- | --- |
| `register` | `register(string $slug, array $manifest): void` | Register a theme from its manifest (metadata, layouts, regions, assets, settings schema). |
| `layouts` | `layouts(string $slug): array` | List the layout definitions declared by the theme. |
| `regions` | `regions(string $layout): array` | List the named, widget-droppable regions of a layout. |
| `assets` | `assets(string $slug): AssetBundle` | Return the compiled CSS/JS `AssetBundle` (versioned, fingerprinted) for the theme. |

## Goco\SDK\Plugin

`namespace Goco\SDK` · **stable** · deep dive: [Plugin SDK](../sdk/plugin-sdk.md), [Plugin Engine](../core/plugin-engine.md)

Register plugins and manage their lifecycle, routes, and permissions.

| Method | Signature | Description |
| --- | --- | --- |
| `register` | `register(string $slug, array $manifest): void` | Register a plugin and its manifest with the plugin engine. |
| `install` | `install(string $slug): void` | Run install migrations, seed data, and create indexes for the plugin. |
| `boot` | `boot(string $slug): void` | Boot an installed plugin — wire hooks, services, and routes into the running app. |
| `routes` | `routes(callable $registrar): void` | Register plugin HTTP routes via a registrar callback receiving the router. |
| `permissions` | `permissions(array $caps): void` | Declare capabilities (`resource.action` strings) the plugin contributes to RBAC. |

```php
use Goco\SDK\Plugin;

Plugin::permissions(['newsletter.send', 'newsletter.subscribers.manage']);
Plugin::routes(function ($router) {
    $router->nsRoute('newsletter', '/subscribe', fn($request) => /* ... */ []);
});
```

## Goco\SDK\Hook

`namespace Goco\SDK` · **stable** · deep dive: [Hook SDK](../sdk/hook-sdk.md), [Event & Hook System](../architecture/event-hook-system.md)

Actions (side-effect events) and filters (value transformers). Actions use `subject.verb[.tense]`; filters use `subject.noun`.

| Method | Signature | Description |
| --- | --- | --- |
| `listen` | `listen(string $action, callable $cb, int $priority = 10): void` | Subscribe to an action; lower priority runs earlier. Alias: `on()`. |
| `dispatch` | `dispatch(string $action, mixed ...$args): void` | Fire an action synchronously to all listeners. Alias: `do()`. |
| `dispatchAsync` | `dispatchAsync(string $action, mixed ...$args): void` | Fire an action on a background coroutine/queue without blocking the request. |
| `filter` | `filter(string $name, callable $cb, int $priority = 10): void` | Register a filter that transforms a value. |
| `apply` | `apply(string $name, mixed $value, mixed ...$args): mixed` | Run all filters for `$name` over `$value` and return the result. |

```php
use Goco\SDK\Hook;

Hook::listen('content.published', fn($page) => Cache::tags(['pages'])->flush());
Hook::filter('page.title', fn(string $title) => trim($title) . ' — Acme');
```

## Goco\Container\Container

`namespace Goco\Container` · **stable** · deep dive: [Service Container & DI](../architecture/service-container.md)

PSR-11 compatible container with autowiring, coroutine-scoped bindings, and lazy singletons.

| Method | Signature | Description |
| --- | --- | --- |
| `bind` | `bind(string $id, callable\|string $concrete): void` | Register a transient factory or class binding. |
| `singleton` | `singleton(string $id, callable\|string $concrete): void` | Register a lazily-built, worker-shared singleton. |
| `scoped` | `scoped(string $id, callable\|string $concrete): void` | Register a per-request (per-coroutine) binding. |
| `get` | `get(string $id): mixed` | Resolve a binding, autowiring constructor dependencies. |
| `has` | `has(string $id): bool` | Report whether an id is bound or resolvable. |
| `make` | `make(string $class, array $args = []): mixed` | Instantiate a class with autowiring plus explicit overrides. |
| `call` | `call(callable $target, array $args = []): mixed` | Invoke a callable, injecting parameters by type/name. |
| `extend` | `extend(string $id, callable $decorator): void` | Decorate an existing binding at resolve time. |

## Goco\Http\Router

`namespace Goco\Http` · **stable** · deep dive: [Routing](../core/routing.md)

Thin, ergonomic wrapper over ZealPHP's routing (`route`, `nsRoute`, `nsPathRoute`, `patternRoute`) with Flask-style params and reflection injection.

| Method | Signature | Description |
| --- | --- | --- |
| `route` | `route(string $path, callable $handler, array $methods = ['GET']): void` | Register a route; `{param}` segments inject by name. |
| `nsRoute` | `nsRoute(string $ns, string $path, callable $handler, array $methods = ['GET']): void` | Register a namespaced route (prefixed path + name isolation). |
| `nsPathRoute` | `nsPathRoute(string $ns, string $path, callable $handler): void` | Register a namespaced catch-all path route. |
| `patternRoute` | `patternRoute(string $regex, callable $handler): void` | Register a route matched by regular expression. |
| `group` | `group(array $attributes, callable $routes): void` | Apply shared prefix/middleware/name to a block of routes. |
| `middleware` | `middleware(MiddlewareInterface ...$mw): static` | Attach PSR-15 middleware to subsequently-registered routes. |
| `redirect` | `redirect(string $from, string $to, int $status = 302): void` | Register an HTTP redirect route. |

> **Tip**
> Handlers may return `int|array|string|Generator`. Arrays become JSON, generators stream, ints set the status code. See [Request Lifecycle](../architecture/request-lifecycle.md).

## Goco\Http Request & Response helpers

`namespace Goco\Http` · **stable** · deep dive: [Request Lifecycle](../architecture/request-lifecycle.md)

Coroutine-isolated wrappers over the OpenSwoole request/response, plus PSR-7-style accessors.

### `Request`

| Method | Signature | Description |
| --- | --- | --- |
| `input` | `input(string $key, mixed $default = null): mixed` | Read a merged query/body parameter. |
| `query` | `query(?string $key = null, mixed $default = null): mixed` | Read query-string parameters. |
| `json` | `json(?string $key = null): mixed` | Decode and read the JSON body. |
| `file` | `file(string $key): ?UploadedFile` | Access an uploaded file. |
| `header` | `header(string $name, ?string $default = null): ?string` | Read a request header. |
| `bearerToken` | `bearerToken(): ?string` | Extract the `Authorization: Bearer` token. |
| `user` | `user(): ?Auth\Identity` | Return the authenticated identity, if any. |
| `ip` | `ip(): string` | Return the client IP (Traefik `X-Forwarded-For` aware). |
| `validate` | `validate(array $rules): array` | Validate input against rules; throws `ValidationException` on failure. |

### `Response`

| Method | Signature | Description |
| --- | --- | --- |
| `json` | `json(mixed $data, int $status = 200): Response` | Emit a JSON response. |
| `text` | `text(string $body, int $status = 200): Response` | Emit a plain-text response. |
| `html` | `html(string $body, int $status = 200): Response` | Emit an HTML response. |
| `stream` | `stream(Generator $chunks): Response` | Stream a generator body chunk-by-chunk. |
| `sse` | `sse(): Response` | Switch the connection into Server-Sent Events mode. |
| `header` | `header(string $name, string $value): Response` | Set a response header (chainable). |
| `cookie` | `cookie(string $name, string $value, array $options = []): Response` | Set a cookie. |
| `status` | `status(int $code): Response` | Set the HTTP status code. |
| `redirect` | `redirect(string $url, int $status = 302): Response` | Emit a redirect response. |

## Goco\Storage\StorageDriver

`namespace Goco\Storage` · **stable** · deep dive: [Storage & Media](../architecture/storage.md)

Driver interface for object storage. Concrete drivers: `LocalDriver`, `MinioDriver`, `S3Driver`. Select via `STORAGE_DRIVER` env.

| Method | Signature | Description |
| --- | --- | --- |
| `put` | `put(string $path, string\|resource $contents, array $options = []): string` | Store an object; returns its canonical key. |
| `get` | `get(string $path): string` | Read an object's contents. |
| `stream` | `stream(string $path): resource` | Open a read stream for large objects. |
| `exists` | `exists(string $path): bool` | Check whether an object exists. |
| `delete` | `delete(string $path): bool` | Delete an object (soft-delete aware for media). |
| `url` | `url(string $path): string` | Return a public URL for an object. |
| `temporaryUrl` | `temporaryUrl(string $path, int $ttl = 3600): string` | Return a signed, expiring URL. |
| `copy` | `copy(string $from, string $to): bool` | Copy an object within the driver. |
| `size` | `size(string $path): int` | Return the object size in bytes. |

## Goco\Search\SearchProvider

`namespace Goco\Search` · **beta** · deep dive: [Search](../architecture/search.md)

Swappable search provider interface. Concrete providers: `MongoTextProvider`, `AtlasSearchProvider`, `MeilisearchProvider`, `OpenSearchProvider`. Select via `SEARCH_PROVIDER` env.

| Method | Signature | Description |
| --- | --- | --- |
| `index` | `index(string $collection, array $document): void` | Add or update a document in the search index. |
| `bulk` | `bulk(string $collection, iterable $documents): void` | Index documents in batches. |
| `delete` | `delete(string $collection, string $id): void` | Remove a document from the index. |
| `search` | `search(string $collection, string $query, array $options = []): SearchResult` | Execute a query with filters, facets, pagination. |
| `suggest` | `suggest(string $collection, string $prefix, int $limit = 10): array` | Return typeahead suggestions. |
| `configure` | `configure(string $collection, array $settings): void` | Set searchable/filterable/sortable attributes and synonyms. |
| `flush` | `flush(string $collection): void` | Drop and rebuild the index for a collection. |

## Goco\Database — Repositories & Query

`namespace Goco\Database` · **stable** · deep dive: [MongoDB Data Layer](../architecture/database-mongodb.md), [Data Model](../architecture/data-model.md)

A lightweight document-mapper + Repository pattern over MongoDB — not a heavy ORM. Every repository is workspace/website scoped and soft-delete aware.

### `Repository<T>`

| Method | Signature | Description |
| --- | --- | --- |
| `find` | `find(string $id): ?Document` | Fetch one document by `_id` (excludes soft-deleted). |
| `findBy` | `findBy(array $criteria): ?Document` | Fetch the first document matching criteria. |
| `all` | `all(array $criteria = [], array $options = []): Cursor` | Return a lazy cursor of matching documents. |
| `paginate` | `paginate(array $criteria, int $page = 1, int $perPage = 20): Paginator` | Return a paginated result set. |
| `create` | `create(array $attributes): Document` | Insert a document, stamping audit + tenant fields. |
| `update` | `update(string $id, array $attributes): Document` | Update a document and bump `version`. |
| `delete` | `delete(string $id): bool` | Soft-delete (set `deleted_at`). |
| `forceDelete` | `forceDelete(string $id): bool` | Permanently remove a document. |
| `restore` | `restore(string $id): bool` | Clear `deleted_at` on a soft-deleted document. |
| `withTrashed` | `withTrashed(): static` | Include soft-deleted documents in subsequent queries. |
| `transaction` | `transaction(callable $work): mixed` | Run `$work` inside a multi-document transaction. |
| `aggregate` | `aggregate(array $pipeline): Cursor` | Execute an aggregation pipeline. |
| `count` | `count(array $criteria = []): int` | Count matching documents. |

### `QueryBuilder`

| Method | Signature | Description |
| --- | --- | --- |
| `where` | `where(string $field, mixed $op, mixed $value = null): static` | Add a filter clause. |
| `whereIn` | `whereIn(string $field, array $values): static` | Match any value in a set. |
| `orderBy` | `orderBy(string $field, string $dir = 'asc'): static` | Add a sort. |
| `limit` | `limit(int $n): static` | Limit the result count. |
| `skip` | `skip(int $n): static` | Skip documents (offset). |
| `select` | `select(string ...$fields): static` | Projection — return only named fields. |
| `get` | `get(): Cursor` | Execute and return a cursor. |
| `first` | `first(): ?Document` | Execute and return the first match. |

> **Note**
> Repositories automatically inject `workspace_id` and `website_id` predicates from the active [Context](../architecture/multi-tenancy.md). Use `withTrashed()` explicitly when you need soft-deleted rows.

## Goco\Config\Config

`namespace Goco\Config` · **stable** · deep dive: [Configuration Reference](configuration-reference.md), [Configuration](../getting-started/configuration.md)

Typed, env-backed configuration accessor. Values resolve from `.env`, compiled config files, and per-workspace `settings`.

| Method | Signature | Description |
| --- | --- | --- |
| `get` | `get(string $key, mixed $default = null): mixed` | Read a dotted config key. |
| `set` | `set(string $key, mixed $value): void` | Override a config value at runtime (worker-local). |
| `has` | `has(string $key): bool` | Check whether a key is defined. |
| `env` | `env(string $key, mixed $default = null): mixed` | Read a raw environment variable. |
| `bool` / `int` / `string` | `bool(string $key, bool $default = false): bool` | Typed accessors with coercion. |
| `forWebsite` | `forWebsite(string $websiteId): ConfigScope` | Return a config scope merged with a website's `settings`. |

## Goco\Cache\Cache

`namespace Goco\Cache` · **stable** · deep dive: [Caching, Queue & Realtime](../architecture/caching-and-queue.md)

Redis-backed cache with tags, locks, and atomic counters.

| Method | Signature | Description |
| --- | --- | --- |
| `get` | `get(string $key, mixed $default = null): mixed` | Read a cached value. |
| `put` | `put(string $key, mixed $value, int $ttl = 3600): bool` | Store a value with a TTL (seconds). |
| `remember` | `remember(string $key, int $ttl, callable $cb): mixed` | Return cached value or compute, store, and return it. |
| `forget` | `forget(string $key): bool` | Delete a key. |
| `tags` | `tags(array $tags): TaggedCache` | Return a tagged cache scope for grouped invalidation. |
| `lock` | `lock(string $name, int $ttl = 10): Lock` | Acquire a distributed lock. |
| `increment` | `increment(string $key, int $by = 1): int` | Atomically increment a counter. |
| `flush` | `flush(): bool` | Clear the active cache store. |

## Goco\Queue\Queue

`namespace Goco\Queue` · **stable** · deep dive: [Caching, Queue & Realtime](../architecture/caching-and-queue.md)

Redis-backed job queue with delays, retries, and backoff. Jobs are persisted in the `jobs` collection for durability and audit.

| Method | Signature | Description |
| --- | --- | --- |
| `push` | `push(Job $job, ?string $queue = null): string` | Enqueue a job; returns a job id. |
| `later` | `later(int $delay, Job $job, ?string $queue = null): string` | Enqueue a job to run after `$delay` seconds. |
| `bulk` | `bulk(iterable $jobs, ?string $queue = null): array` | Enqueue many jobs at once. |
| `dispatch` | `dispatch(callable $work, array $options = []): string` | Enqueue an inline closure job. |
| `onQueue` | `onQueue(string $queue): static` | Target a named queue. |
| `release` | `release(Job $job, int $delay = 0): void` | Requeue a job for retry with delay. |
| `size` | `size(?string $queue = null): int` | Return pending job count. |

## Goco\Auth\Auth

`namespace Goco\Auth` · **stable** · deep dive: [Authentication](../core/authentication.md), [Permission System](../architecture/permission-system.md), [Security Model](../security/security-model.md)

Authentication and authorization facade. Sessions live in Redis; API auth uses JWT. Supports OAuth2, 2FA (TOTP), and Passkeys (WebAuthn). Passwords hashed with Argon2id.

| Method | Signature | Description |
| --- | --- | --- |
| `attempt` | `attempt(array $credentials, bool $remember = false): ?Identity` | Verify credentials and start a session. |
| `user` | `user(): ?Identity` | Return the current authenticated identity. |
| `id` | `id(): ?string` | Return the current user id. |
| `check` | `check(): bool` | Report whether a user is authenticated. |
| `login` | `login(Identity $user, bool $remember = false): void` | Establish a session for an identity. |
| `logout` | `logout(): void` | Destroy the current session. |
| `can` | `can(string $capability, mixed $resource = null): bool` | RBAC/ABAC capability check (`resource.action`). |
| `authorize` | `authorize(string $capability, mixed $resource = null): void` | Assert a capability; throws `AuthorizationException`. |
| `hasRole` | `hasRole(string $role): bool` | Check role membership within the active scope. |
| `issueToken` | `issueToken(Identity $user, array $claims = [], int $ttl = 3600): string` | Mint a signed JWT for API use. |
| `verifyToken` | `verifyToken(string $jwt): ?Identity` | Validate a JWT and resolve its identity. |

## Goco\Cli\Command

`namespace Goco\Cli` · **stable** · deep dive: [CLI SDK](../sdk/cli.md), [CLI Reference](cli-reference.md)

Base class for `goco` console commands (the developer CLI: lifecycle + generators). Extend and register via a plugin or package.

| Member | Signature | Description |
| --- | --- | --- |
| `$signature` | `protected string $signature` | Command name + argument/option definition (e.g. `make:widget {name}`). |
| `$description` | `protected string $description` | One-line help text. |
| `handle` | `handle(): int` | Execute the command; return an exit code. |
| `argument` | `argument(string $name): mixed` | Read a positional argument. |
| `option` | `option(string $name): mixed` | Read an option/flag. |
| `ask` | `ask(string $question, ?string $default = null): string` | Prompt for input. |
| `confirm` | `confirm(string $question, bool $default = false): bool` | Prompt for yes/no. |
| `info` / `warn` / `error` | `info(string $message): void` | Write styled output to the console. |
| `table` | `table(array $headers, array $rows): void` | Render a table. |
| `call` | `call(string $command, array $args = []): int` | Invoke another registered command. |

---

# Part B — HTTP API

Served by `apps/api` under the base path **`/api/v1`**, published through Traefik with automatic HTTPS. The API offers a **REST** surface (resource-oriented, JSON) and a co-located **GraphQL** endpoint for aggregate queries. Deep dive per resource is linked inline; the transport rules below apply to every endpoint.

## Base URL & versioning

```
https://{tenant-domain}/api/v1
```

- Version is pinned in the path (`/api/v1`). Breaking changes ship as `/api/v2`; additive changes stay in `v1`.
- Tenancy is resolved from the request host (Traefik per-tenant routers) and/or the `X-Workspace` / `X-Website` headers; the resolved `workspace_id` + `website_id` scope every query. See [Multi-Tenancy](../architecture/multi-tenancy.md).
- `Accept: application/json` is assumed. GraphQL uses `Content-Type: application/json`.

## Authentication

| Scheme | Header | Use |
| --- | --- | --- |
| JWT (Bearer) | `Authorization: Bearer <jwt>` | Primary machine + SPA auth. Obtain via `POST /api/v1/auth/token`. |
| OAuth2 | `Authorization: Bearer <access_token>` | Third-party/marketplace apps (authorization-code + client-credentials grants). |
| Session | Cookie + `X-CSRF-Token` | First-party browser clients (admin app). CSRF enforced by the ZealPHP `Csrf` middleware. |

```bash
curl -X POST https://acme.example.com/api/v1/auth/token \
  -H 'Content-Type: application/json' \
  -d '{"email":"editor@acme.com","password":"••••••","totp":"123456"}'
# -> { "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 3600, "refresh_token": "..." }
```

Every request is authorized against the caller's capabilities (`resource.action`) within the resolved scope. A missing capability returns `403`. See [Authentication](../core/authentication.md) and [Permission System](../architecture/permission-system.md).

## Pagination, filtering & sorting

Collection endpoints are cursor- or page-paginated:

| Parameter | Example | Description |
| --- | --- | --- |
| `page`, `per_page` | `?page=2&per_page=50` | Offset pagination (default `per_page=20`, max `100`). |
| `cursor` | `?cursor=eyJpZCI6...` | Opaque cursor for stable deep pagination. |
| `sort` | `?sort=-created_at` | Sort field; `-` prefix = descending. |
| `filter[field]` | `?filter[status]=published` | Field filters (equality, `in`, ranges via `filter[views][gte]=100`). |
| `q` | `?q=welcome` | Full-text query routed to the active [search provider](../architecture/search.md). |
| `fields` | `?fields=id,title,slug` | Sparse fieldsets (projection). |
| `include` | `?include=author,revisions` | Embed related resources. |

```json
{
  "data": [ { "id": "665f...", "title": "Welcome", "status": "published" } ],
  "meta": { "page": 1, "per_page": 20, "total": 137, "total_pages": 7 },
  "links": { "next": "/api/v1/pages?page=2", "prev": null }
}
```

## Error format (RFC 7807)

```json
{
  "type": "https://gococms.org/errors/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "The 'slug' field is required.",
  "errors": { "slug": ["required"] },
  "trace_id": "01J9X..."
}
```

| Status | Meaning |
| --- | --- |
| `400` | Malformed request. |
| `401` | Missing/invalid credentials. |
| `403` | Authenticated but lacks the required capability. |
| `404` | Resource not found (or soft-deleted). |
| `409` | Version conflict (optimistic concurrency on `version`). |
| `422` | Validation error (`errors` map). |
| `429` | Rate limited (see below). |
| `5xx` | Server error; `trace_id` correlates with `audit_logs` and logs in `/tmp/zealphp/`. |

## Rate limiting

Enforced by the ZealPHP `RateLimit` middleware backed by Redis, keyed per identity + route class.

| Header | Description |
| --- | --- |
| `X-RateLimit-Limit` | Requests allowed in the window. |
| `X-RateLimit-Remaining` | Requests left in the current window. |
| `X-RateLimit-Reset` | Unix epoch when the window resets. |
| `Retry-After` | Seconds to wait (on `429`). |

Defaults: `600/min` authenticated, `60/min` anonymous, `10/min` for `auth/*`. Tunable per plan/tenant — see [Configuration Reference](configuration-reference.md).

## REST endpoints

Representative endpoints per resource. `{ws}`/`{web}` are implied by scope resolution.

### Auth — deep dive: [Authentication](../core/authentication.md)

| Method & Path | Capability | Description |
| --- | --- | --- |
| `POST /api/v1/auth/token` | — | Exchange credentials (+ optional TOTP) for a JWT + refresh token. |
| `POST /api/v1/auth/refresh` | — | Rotate an access token using a refresh token. |
| `POST /api/v1/auth/logout` | — | Revoke the current token/session. |
| `GET /api/v1/auth/me` | — | Return the current identity, roles, and capabilities. |
| `POST /api/v1/auth/passkeys/challenge` | — | Begin a WebAuthn (passkey) ceremony. |
| `POST /api/v1/auth/2fa/verify` | — | Verify a TOTP code. |

### Pages — deep dive: [Page Builder](../core/page-builder.md)

| Method & Path | Capability | Description |
| --- | --- | --- |
| `GET /api/v1/pages` | `pages.read` | List pages (filter by status, template, parent). |
| `POST /api/v1/pages` | `pages.create` | Create a page. |
| `GET /api/v1/pages/{id}` | `pages.read` | Fetch a page with layout tree. |
| `PATCH /api/v1/pages/{id}` | `pages.update` | Partially update a page (optimistic `version`). |
| `DELETE /api/v1/pages/{id}` | `pages.delete` | Soft-delete a page. |
| `POST /api/v1/pages/{id}/publish` | `pages.publish` | Publish a page; fires `content.published`. |
| `GET /api/v1/pages/{id}/revisions` | `pages.read` | List `page_revisions`. |
| `POST /api/v1/pages/{id}/restore/{rev}` | `pages.update` | Restore a revision. |

### Posts — deep dive: [Blog Engine](../core/blog-engine.md)

| Method & Path | Capability | Description |
| --- | --- | --- |
| `GET /api/v1/posts` | `posts.read` | List posts (filter by taxonomy/term, author, status). |
| `POST /api/v1/posts` | `posts.create` | Create a post. |
| `GET /api/v1/posts/{id}` | `posts.read` | Fetch a post. |
| `PATCH /api/v1/posts/{id}` | `posts.update` | Update a post. |
| `DELETE /api/v1/posts/{id}` | `posts.delete` | Soft-delete a post. |
| `POST /api/v1/posts/{id}/publish` | `posts.publish` | Publish a post. |
| `GET /api/v1/taxonomies/{tax}/terms` | `posts.read` | List terms in a taxonomy. |

### Media — deep dive: [Storage & Media](../architecture/storage.md)

| Method & Path | Capability | Description |
| --- | --- | --- |
| `GET /api/v1/media` | `media.read` | List media assets. |
| `POST /api/v1/media` | `media.create` | Upload an asset (multipart) to the active storage driver. |
| `POST /api/v1/media/presign` | `media.create` | Get a signed direct-upload URL (MinIO/S3). |
| `GET /api/v1/media/{id}` | `media.read` | Fetch asset metadata + variants. |
| `DELETE /api/v1/media/{id}` | `media.delete` | Soft-delete an asset. |

### Collections (Database Builder) — deep dive: [Database Builder](../core/database-builder.md)

Dynamic, user-defined collections addressed by slug.

| Method & Path | Capability | Description |
| --- | --- | --- |
| `GET /api/v1/collections` | `collections.manage` | List collection definitions. |
| `GET /api/v1/collections/{slug}` | `collections.read` | Fetch a collection's schema. |
| `GET /api/v1/collections/{slug}/entries` | `collections.read` | List entries (filter/sort/paginate). |
| `POST /api/v1/collections/{slug}/entries` | `collections.create` | Create an entry (validated against JSON-Schema). |
| `GET /api/v1/collections/{slug}/entries/{id}` | `collections.read` | Fetch one entry. |
| `PATCH /api/v1/collections/{slug}/entries/{id}` | `collections.update` | Update an entry. |
| `DELETE /api/v1/collections/{slug}/entries/{id}` | `collections.delete` | Soft-delete an entry. |

### Users & roles — deep dive: [Permission System](../architecture/permission-system.md)

| Method & Path | Capability | Description |
| --- | --- | --- |
| `GET /api/v1/users` | `users.manage` | List users in the workspace. |
| `POST /api/v1/users` | `users.manage` | Invite/create a user. |
| `PATCH /api/v1/users/{id}` | `users.manage` | Update a user (roles, profile). |
| `DELETE /api/v1/users/{id}` | `users.manage` | Deactivate a user. |
| `GET /api/v1/roles` | `users.manage` | List roles and their capabilities. |

### Plugins — deep dive: [Plugin Engine](../core/plugin-engine.md), [Marketplace](../marketplace/overview.md)

| Method & Path | Capability | Description |
| --- | --- | --- |
| `GET /api/v1/plugins` | `plugins.manage` | List installed plugins and status. |
| `POST /api/v1/plugins` | `plugins.manage` | Install a plugin from the marketplace or package. |
| `POST /api/v1/plugins/{slug}/activate` | `plugins.manage` | Activate a plugin; fires `plugin.activated`. |
| `POST /api/v1/plugins/{slug}/deactivate` | `plugins.manage` | Deactivate a plugin. |
| `DELETE /api/v1/plugins/{slug}` | `plugins.manage` | Uninstall a plugin. |

### Themes — deep dive: [Theme Engine](../core/theme-engine.md)

| Method & Path | Capability | Description |
| --- | --- | --- |
| `GET /api/v1/themes` | `themes.manage` | List available/installed themes. |
| `POST /api/v1/themes/{slug}/activate` | `themes.manage` | Activate a theme for the website. |
| `GET /api/v1/themes/{slug}/layouts` | `themes.manage` | List a theme's layouts and regions. |
| `PATCH /api/v1/websites/{id}/theme-settings` | `themes.manage` | Update theme settings. |

### Search — deep dive: [Search](../architecture/search.md)

| Method & Path | Capability | Description |
| --- | --- | --- |
| `GET /api/v1/search` | scope-dependent | Cross-collection search via the active provider (`?q=&filter=&facets=`). |
| `GET /api/v1/search/suggest` | scope-dependent | Typeahead suggestions. |

## GraphQL

A single endpoint co-located with the REST API for aggregate reads and mutations:

```
POST /api/v1/graphql
```

Same JWT/OAuth auth, tenancy scoping, and capability checks as REST. Introspection is enabled in non-production only.

```graphql
query LatestPosts {
  posts(filter: { status: PUBLISHED }, sort: "-published_at", first: 5) {
    edges {
      node { id title slug author { name } }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

> **Warning**
> GraphQL is `beta`. Query depth and complexity are capped, and persisted queries are recommended for public clients. Field-level authorization mirrors REST capabilities.

## Webhooks & realtime

- **Webhooks** — subscribe to hook actions (`content.published`, `form_submissions.created`, `plugin.activated`) delivered as signed `POST`s. Managed under `POST /api/v1/webhooks`.
- **Realtime** — a WebSocket endpoint (`wss://{host}/ws`) and SSE streams (`GET /api/v1/stream/...`) push live updates, backed by Redis pub/sub. See [Caching, Queue & Realtime](../architecture/caching-and-queue.md).

---

## Related

- [Widget SDK](../sdk/widget-sdk.md)
- [Theme SDK](../sdk/theme-sdk.md)
- [Plugin SDK](../sdk/plugin-sdk.md)
- [Hook SDK](../sdk/hook-sdk.md)
- [CLI SDK](../sdk/cli.md)
- [CLI Reference](cli-reference.md)
- [Configuration Reference](configuration-reference.md)
- [Service Container & Dependency Injection](../architecture/service-container.md)
- [Event & Hook System](../architecture/event-hook-system.md)
- [MongoDB Data Layer](../architecture/database-mongodb.md)
- [Permission System (RBAC + ABAC)](../architecture/permission-system.md)
- [Security Model](../security/security-model.md)
- [Routing](../core/routing.md)
- [Documentation Index](../README.md)
