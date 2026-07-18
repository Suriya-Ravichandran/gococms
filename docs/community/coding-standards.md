# Coding Standards

> The engineering rules every GOCO CMS contribution must follow: PSR-12/PSR-4 PHP, PHP 8.4 idioms, coroutine-safe async code, repository-mediated MongoDB access, static analysis at max level, and Conventional Commits.

These standards are enforced by CI. A pull request that violates them will fail the pipeline before review. They exist so that a lightweight core surrounded by a large ecosystem of widgets, themes, and plugins stays consistent, safe under OpenSwoole coroutines, and maintainable across dozens of packages in the monorepo.

Applies to `gococms/core`, `gococms/cli`, every `packages/*`, and every first-party plugin, theme, and widget. Third-party marketplace authors are strongly encouraged to adopt the same rules; the `goco lint` command ships the exact configuration.

---

## 1. Language & Tooling Baseline

| Concern | Standard | Enforced by |
|---|---|---|
| Runtime | PHP 8.4+ on OpenSwoole 22.1+ (ZealPHP) | `composer.json` `require`, CI matrix |
| Style | PSR-12 (superset of PSR-1/PSR-2) | php-cs-fixer, PHP_CodeSniffer |
| Autoloading | PSR-4, one class per file | Composer, PHPStan |
| Static analysis | PHPStan level `max` and Psalm level `1` | CI (`goco lint:types`) |
| Formatting | php-cs-fixer with the shipped ruleset | CI (`goco lint:style`) |
| Tests | PHPUnit + coroutine harness | CI (`goco test`) |
| Commits | Conventional Commits + SemVer | commitlint, CI |
| Docblocks | Required on all public API surfaces | PHPStan `--strict`, review |

> **Note**
> Run the full local gate with a single command before pushing:
> ```bash
> goco lint && goco test
> ```
> `goco lint` runs php-cs-fixer (fix), PHPStan, and Psalm; `goco test` runs unit, integration, and coroutine suites. Both use the same configuration CI uses, so a green local run predicts a green pipeline.

---

## 2. PHP Style — PSR-12 & PSR-4

All code follows [PSR-12](https://www.php-fig.org/psr/psr-12/). The non-negotiable points:

- 4-space indentation, no tabs. Unix `LF` line endings. UTF-8 without BOM. Files end with a single newline.
- Opening brace of classes and methods on their own line; opening brace of control structures on the same line.
- One blank line after the `namespace` declaration and after the `use` block.
- Soft limit of 120 columns; hard limit of 160. Wrap long argument lists one-per-line.
- Visibility declared on every property and method. `abstract`/`final` before visibility; `static` after.

PSR-4 autoloading: the namespace root `Goco\` maps to `core/src/`, and each package maps its own `src/` (for example `Goco\Database\` -> `packages/database/src/`). One public class, interface, enum, or trait per file, and the file name must match the type name exactly.

```php
<?php

declare(strict_types=1);

namespace Goco\Widget;

use Goco\SDK\Context;
use Goco\Widget\Contract\WidgetRenderer;

final class HeroWidget implements WidgetRenderer
{
    public function render(array $props, ?Context $ctx = null): string
    {
        // ...
    }
}
```

> **Tip**
> Do not hand-format. Run `php-cs-fixer` (via `goco lint:style --fix`) and let it own whitespace, import ordering, and brace placement. Review comments about style should be rare because the formatter is authoritative.

---

## 3. PHP 8.4 Language Features

GOCO targets PHP 8.4, so use the modern language directly. The following are expected, not optional.

**`declare(strict_types=1)` is mandatory** as the first statement of every PHP file. No exceptions. It prevents silent scalar coercion, which is a frequent source of subtle bugs in a document-oriented system where values cross MongoDB, Redis, and HTTP boundaries.

**Type everything.** Typed properties, typed parameters, typed return values including `void` and `never`. Prefer union and intersection types over `mixed`. `mixed` requires a docblock explaining the shape.

```php
public function priority(): int { /* ... */ }
public function handle(Request $request): Response { /* ... */ }
private function assertNever(): never { throw new \LogicException(); }
```

**`readonly` for value objects and DTOs.** Anything that models an immutable fact — a rendered context, a property schema, an event payload — should be `readonly`. Use constructor property promotion.

```php
final readonly class WidgetContext
{
    public function __construct(
        public string $workspaceId,
        public string $websiteId,
        public string $locale,
        public array $props = [],
    ) {}
}
```

**Enums instead of class constants for closed sets.** Backed enums for anything persisted or serialized.

```php
enum PublishState: string
{
    case Draft     = 'draft';
    case Scheduled = 'scheduled';
    case Published  = 'published';
    case Archived   = 'archived';

    public function isVisible(): bool
    {
        return $this === self::Published;
    }
}
```

**First-class callable syntax** (`$this->render(...)`), **named arguments** for optional/boolean flags, **`match`** over `switch` for expression mapping, and **nullsafe** operators where they clarify intent. Use `#[\Override]` on overriding methods. Use property hooks and asymmetric visibility (`public private(set)`) where they replace boilerplate getters/setters.

> **Warning**
> Enums, `readonly` objects, and promoted properties are especially valuable under OpenSwoole because immutable values are inherently safe to read across coroutines. Prefer them wherever a piece of state is computed once and read many times. See [Async & Coroutine Safety](#5-async--coroutine-safety-critical).

---

## 4. Naming, Docblocks, Exceptions

### 4.1 Naming conventions

| Element | Convention | Example |
|---|---|---|
| Namespace / class / interface / enum / trait | `StudlyCaps` | `TemplateEngine`, `WidgetRenderer` |
| Interface | noun or `-able`, no `I` prefix | `Repository`, `Cacheable` |
| Method / function / variable | `camelCase` | `renderToString()`, `$pageId` |
| Constant / enum case | `SCREAMING_SNAKE` / `StudlyCaps` | `MODE_COROUTINE`, `State::Draft` |
| MongoDB collection | `snake_case` plural | `page_revisions`, `form_submissions` |
| Document field | `snake_case` | `workspace_id`, `created_at` |
| Hook action | `subject.verb[.tense]` | `content.publishing`, `page.rendered` |
| Hook filter | `subject.noun` | `page.title`, `menu.items` |
| Config / env key | `SCREAMING_SNAKE` | `GOCO_MONGO_URI` |

Name by domain intent, not implementation. `PageRepository`, not `PageMongoDao`. Booleans read as predicates: `isPublished`, `hasCapability`, `canPublish`.

### 4.2 Docblocks

Every **public** class, method, and function that is part of an API surface (SDK facades, package public interfaces, plugin extension points) carries a docblock. The docblock adds what the signature cannot: array shapes, thrown exceptions, generic types, stability, and units.

```php
/**
 * Render a registered widget to an HTML string.
 *
 * @param array<string, mixed> $props Validated against the widget's PropertySchema.
 * @param Context|null $ctx Request-scoped context; null uses the ambient RequestContext.
 * @return string Escaped, cache-key-stable HTML fragment.
 *
 * @throws WidgetNotRegisteredException When $type has no registration.
 * @throws PropertyValidationException  When $props fail the schema.
 *
 * @stable
 */
public static function render(string $type, array $props, ?Context $ctx = null): string
```

Rules:

- Use `@stable`, `@beta`, `@experimental`, or `@deprecated` on public API. `@deprecated` must name the replacement and the removal version.
- Use PHPStan/psalm array shapes (`array{id: string, title: string}`) and generics (`@return list<Page>`) rather than bare `array`.
- Do **not** write docblocks that merely restate the signature. `@param int $id The id` is noise; delete it.
- Private/internal methods need docblocks only when the type system is insufficient.

### 4.3 Exceptions & error handling

- Throw typed, domain-specific exceptions from a package's `Exception\` namespace. Each package defines a marker interface (for example `Goco\Database\Exception\DatabaseException`) so callers can catch by domain.
- Extend the correct SPL base: `InvalidArgumentException` for bad input, `RuntimeException` for runtime failures, `LogicException` for programmer errors that indicate a bug.
- Never catch `\Throwable` to swallow it. Catch the narrowest type you can handle; rethrow or wrap the rest with context.
- Never return `false`/`null` to signal an error condition that the caller cannot distinguish from a valid value — throw instead.
- Include actionable context in the message (ids, collection names) but **never** secrets, tokens, or full documents.

```php
try {
    $page = $pages->findPublished($workspaceId, $websiteId, $slug);
} catch (PageNotFoundException $e) {
    Hook::dispatch('page.not_found', $slug);
    return $response->status(404);
} // Any other DatabaseException propagates to the global handler.
```

Errors surface through the ZealPHP response lifecycle; the global handler logs to `/tmp/zealphp/` and emits a sanitized problem response. Do not `echo` errors or leak stack traces to clients in production mode.

---

## 5. Async & Coroutine Safety (CRITICAL)

GOCO runs on OpenSwoole in `App::MODE_COROUTINE` by default. A single worker process handles thousands of concurrent requests, each on its own coroutine, sharing the same PHP memory. **Code that is correct under Apache/PHP-FPM can corrupt data or deadlock here.** These rules are the most important in this document.

### 5.1 No blocking I/O on the request path

In coroutine mode, blocking calls stall the entire worker and every coroutine it is multiplexing. Use OpenSwoole-aware, non-blocking clients and coroutine primitives.

- **Forbidden:** `sleep()`, `usleep()`, native blocking `curl`, `file_get_contents()` over the network, blocking `PDO`/`mysqli`, `exec()` of long processes, synchronous DNS.
- **Required:** the MongoDB and Redis clients wired through the service container (coroutine-scheduling), `co::sleep()` / `\OpenSwoole\Coroutine\sleep()` instead of `sleep()`, coroutine HTTP clients, and generator streaming for large responses.

```php
// WRONG — blocks the whole worker
sleep(2);
$html = file_get_contents('https://api.example.com/v1/data');

// RIGHT — yields the coroutine, worker keeps serving others
co::sleep(2);
$html = $httpClient->get('https://api.example.com/v1/data'); // coroutine client
```

> **Warning**
> A single stray `sleep()` or blocking socket in a hot handler degrades the entire worker's throughput to serial. PHPStan ships a custom rule that flags the blocking-function denylist; do not suppress it.

### 5.2 No shared mutable global or static state across requests

The worker process is long-lived, so `static` properties, mutable singletons, and superglobal writes persist **between requests** and are **shared between coroutines**. Request A can read request B's data — a tenant-isolation and security failure.

- Do not store per-request data in `static` properties, class-level caches keyed implicitly by request, or module-level variables.
- `App::superglobals(false)` is set at bootstrap; read request data from the injected `$request`, not from ambient `$_GET`/`$_POST`.
- `$_SESSION` is safe: `ext-zealphp` overrides it with **per-coroutine isolation**. Still call `session_start()` per request.
- A service registered as a singleton in the container must be **stateless** (or hold only immutable config). If it needs per-request data, that data comes in as a method argument, not as instance state.

```php
// WRONG — leaks across requests and coroutines
final class CurrentUser
{
    private static ?User $user = null;         // shared by every coroutine
    public static function set(User $u): void { self::$user = $u; }
}

// RIGHT — request scope owns the value
$user = G::get('user');                        // or RequestContext::current()->user()
```

### 5.3 Request scope via `ZealPHP\G` / `RequestContext`

Per-request state lives in the coroutine-local container, never in class statics.

- Read/write request-scoped values through `\ZealPHP\G` (the request-context accessor) or `RequestContext`. These are isolated per coroutine and torn down when the request completes.
- Store the authenticated user, resolved `workspace_id`/`website_id`, locale, request id, and feature flags here.
- Never cache a `RequestContext` value into a longer-lived scope (a singleton, a `static`, a `Store`). Copying request scope into shared scope reintroduces the leak.

```php
// Set once during request resolution (middleware / router)
G::set('tenant', new Tenant($workspaceId, $websiteId));

// Read anywhere downstream on the same coroutine
$tenant = G::get('tenant');
```

### 5.4 Safe use of `Store` and `Counter`

Cross-worker shared state uses `\ZealPHP\Store` (backed by `OpenSwoole\Table` or Redis) and `\ZealPHP\Counter` (atomic). These are the *only* sanctioned way to share mutable state, and they have rules.

- Use `Store` for genuinely cross-worker/cross-coroutine data (rate-limit buckets, ephemeral pub/sub, small shared registries). Declare its schema up front: `Store::make($name, $size, $cols)`.
- Never treat `Store` as a general cache or database. Persistent state belongs in MongoDB; distributed cache/queue/locks belong in Redis. For multi-node deployments switch to `Store::defaultBackend(Store::BACKEND_REDIS)` so state survives beyond one host.
- Use `Counter` for atomic counters (`new Counter(0)->increment()`); do not emulate one with `Store` read-modify-write, which races.
- `Store` rows are fixed-width — size string columns generously and validate lengths before writing.

```php
Store::make('ratelimit', size: 65536, cols: [['count', Store::INT]]);
Store::set('ratelimit', $ip, ['count' => 0]);
$hits = (new Counter(0))->increment();
```

### 5.5 No long CPU work on the request path

Coroutines yield on I/O, not on CPU. A tight CPU loop (image transforms, large aggregations post-processing, PDF generation, embeddings) blocks the coroutine scheduler and starves concurrent requests.

- Offload heavy or long-running work to **task workers** / the Redis-backed queue (`jobs` collection + queue package). The request enqueues a job and returns; the worker processes it out of band.
- Use `App::onWorkerStart` with `App::tick()`/`App::after()` for periodic/deferred maintenance, not inline loops in handlers.
- For legitimately CPU-bound synchronous sections, dispatch to the OpenSwoole task pool rather than running on the request coroutine.
- Stream large responses with a `Generator` + `co::sleep()` between chunks instead of building a giant string in memory.

```php
// Request path: enqueue and return fast
$queue->push('media.transcode', ['media_id' => $id]);
return $response->status(202)->json(['status' => 'processing']);
```

See [Architecture: Service Container](../architecture/service-container.md) for how coroutine-safe services are constructed and scoped, and the caching/queue and rendering docs for offload patterns.

---

## 6. MongoDB Access Patterns

The data layer in `packages/database` (`Goco\Database`) is a lightweight document-mapper + Repository pattern — **not** a heavy ORM. Follow these access rules.

### 6.1 Repositories only — no ad-hoc queries in controllers

- All persistence goes through a repository (`PageRepository`, `PostRepository`, ...). Controllers, handlers, widgets, and services depend on repository interfaces, never on the raw `MongoDB\Collection`.
- Query construction (filters, projections, aggregation pipelines, indexes) lives inside the repository. This keeps queries testable, index-aware, and consistently tenant-scoped.
- No inline `$collection->find([...])` in a route handler or template. If a query does not exist yet, add a named method to the repository.

```php
// WRONG — raw query in a handler
$app->route('/p/{slug}', function ($slug) use ($mongo) {
    return $mongo->selectCollection('pages')->findOne(['slug' => $slug]); // no tenant scope!
});

// RIGHT — repository method, tenant-scoped, typed
$app->route('/p/{slug}', function ($slug, PageRepository $pages) {
    $tenant = G::get('tenant');
    return $pages->findPublished($tenant->workspaceId, $tenant->websiteId, $slug);
});
```

### 6.2 Always tenant-scoped

Multi-tenant isolation is enforced by `workspace_id` + `website_id` on every tenant-scoped document. Repository methods for tenant collections **must** require and apply both.

- Every query, update, and delete on a tenant collection includes `workspace_id` and `website_id` in its filter. A missing scope is a cross-tenant data-leak bug and will be rejected in review.
- Derive the scope from `RequestContext`/`G`, never from client-supplied input that isn't validated against the resolved tenant.
- Respect soft deletes: default reads filter `deleted_at: null`. Only explicitly-named methods (`findWithTrashed`) see deleted rows.

### 6.3 Documents, indexes, and integrity

- Every document carries `_id`, `created_at`, `updated_at`, `deleted_at`, `version`, `created_by`, `updated_by`; tenant docs add `workspace_id`, `website_id`. Repositories set these — application code does not hand-roll timestamps.
- Ship a JSON-Schema validator with every new collection and document every index it relies on (with the query it serves). Never add a query whose shape has no supporting index in a hot path.
- Use aggregation pipelines for reporting/derived reads, and multi-document **transactions** for cross-collection invariants (for example page + revision + audit log).
- Bump `version` and write to the matching `*_revisions` collection on content mutations; write `audit_logs` for privileged actions.

See [Architecture: MongoDB Data Layer](../architecture/database-mongodb.md) and [Data Model](../architecture/data-model.md) for the collection catalog, validators, and index definitions.

---

## 7. Static Analysis & Formatting

CI runs three gates. All three must pass.

**PHPStan at level `max`.** No `@phpstan-ignore` without an inline reason comment and a linked issue. The custom GOCO ruleset adds the blocking-I/O denylist (§5.1), a static-mutable-state check (§5.2), and a raw-collection-access check (§6.1). Baselines are frozen and shrink-only — you may not grow a baseline to land a change.

**Psalm at level `1`** (strictest) for the packages that opt in, focused on nullability and taint analysis (untrusted input reaching queries, HTML, or the shell).

**php-cs-fixer** with the shipped ruleset (`@PSR12` plus GOCO overrides: ordered imports, no unused imports, native function invocation, strict comparison, trailing commas in multiline). Run `--fix` locally; CI runs `--dry-run --diff` and fails on any diff.

```bash
goco lint:types       # PHPStan (max) + Psalm (level 1)
goco lint:style       # php-cs-fixer dry-run
goco lint:style --fix # apply formatting locally
```

> **Note**
> The exact `phpstan.neon`, `psalm.xml`, and `.php-cs-fixer.dist.php` files live at the monorepo root and are shipped with `gococms/cli`, so first-party packages, plugins, and marketplace authors all lint against identical configuration.

---

## 8. Testing Expectations & Coverage

Tests are part of the change, not a follow-up. See [Testing Strategy](../community/testing-strategy.md) for the full harness, fixtures, and CI matrix; the standards a contribution must meet are:

- **Every bug fix ships a regression test** that fails before the fix and passes after.
- **Every new public method / behavior ships tests.** New SDK surfaces and hooks require both a unit test and an integration test.
- **Coverage floors (enforced):** `packages/*` and `core/` require **>= 85% line coverage**; security-sensitive packages (`auth`, `database`, `plugin-engine`, `storage`) require **>= 90%**. Coverage may not drop on a PR.
- **Coroutine tests run under the OpenSwoole harness.** Concurrency-sensitive code (anything touching `Store`, `Counter`, `G`, sessions, or shared services) must include a test that exercises it across multiple concurrent coroutines to prove no cross-request leakage.
- **MongoDB and Redis tests run against real service containers** (from the Docker compose stack), not mocks, for repository and integration layers. Pure logic is unit-tested without I/O.
- Tests are deterministic and isolated: each test seeds and tears down its own tenant scope; no test depends on another's order or on wall-clock timing.

```bash
goco test                 # full suite
goco test --suite=unit    # fast, no I/O
goco test --coverage      # enforce coverage floors
```

---

## 9. Commit Hygiene & Review

**Conventional Commits**, validated by commitlint in CI and used to compute SemVer bumps (pre-1.0: `feat:` -> minor, `fix:` -> patch, breaking changes noted with `!`/`BREAKING CHANGE:`).

```
feat(widget-engine): add PropertySchema conditional fields
fix(database): scope soft-delete filter to tenant on findMany
docs(sdk): document Hook::apply priority ordering
refactor(auth)!: replace session token format

BREAKING CHANGE: existing sessions are invalidated on upgrade.
```

- Allowed types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`. Scope is a package or app name (`widget-engine`, `admin`, `api`).
- Subject in imperative mood, lower case, no trailing period, <= 72 characters. Body explains *why*, not *what*.
- One logical change per commit. Keep PRs focused and small enough to review; unrelated refactors go in their own PR.
- Rebase onto `main`, keep history linear, no merge commits inside a feature branch. Squash noise before requesting review.
- Every PR: passes `goco lint && goco test`, updates docs for any public API change, and updates the changelog fragment. No red CI, no `@phpstan-ignore` without justification, no lowered coverage.

> **Tip**
> Install the repo git hooks with `goco hooks:install`. The pre-commit hook runs `goco lint:style --fix` on staged files and commitlint on your message, so most violations are caught before they reach CI.

---

## Related

- [Testing Strategy](../community/testing-strategy.md) — test harness, coroutine testing, coverage matrix, CI.
- [Service Container & Dependency Injection](../architecture/service-container.md) — how coroutine-safe services are built and scoped.
- [Contributing](../community/contributing.md) — branch model, PR flow, and review process.
- [Code of Conduct](../community/code-of-conduct.md) — community expectations for collaboration.
- [ZealPHP Foundation](../architecture/zealphp-foundation.md) — runtime modes, coroutines, and the request model.
- [MongoDB Data Layer](../architecture/database-mongodb.md) — repositories, validators, and transactions.
- [Data Model](../architecture/data-model.md) — collections, fields, and indexes.
- [Documentation Index](../README.md)
