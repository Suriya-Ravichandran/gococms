# Hook SDK

> The stable, framework-level API for extending GOCO CMS behavior through **actions** (fire-and-forget events) and **filters** (value transformers) via the `Goco\SDK\Hook` facade.

**Stability:** `stable` · **Namespace:** `Goco\SDK\Hook` · **Package:** `gococms/core`

The Hook SDK is the primary seam through which widgets, themes, and plugins observe and mutate the request lifecycle without touching core code. It exposes two complementary primitives:

- **Actions** — named notifications ("something happened"). Listeners run for side effects; return values are ignored. Dispatched with `Hook::dispatch()`.
- **Filters** — named value pipelines ("transform this value"). Each listener receives the current value and must return the next value. Applied with `Hook::apply()`.

Both are dispatched synchronously on the calling coroutine by default, ordered by an integer **priority**, and identified by dotted, lowercase names. Actions can additionally be offloaded to OpenSwoole task workers with `Hook::dispatchAsync()`.

> **Note** — This page is the developer-facing API reference. For the engine internals (dispatcher architecture, coroutine safety, exception isolation, the complete built-in hook catalog) see [Event & Hook System](../architecture/event-hook-system.md).

---

## API Surface

The entire public surface is a set of static methods on `Goco\SDK\Hook`. All names below are `stable` unless tagged otherwise.

| Method | Alias | Kind | Signature |
| --- | --- | --- | --- |
| `listen` | `on` | action/filter register | `Hook::listen(string $action, callable $cb, int $priority = 10): string` |
| `dispatch` | `do` | action fire | `Hook::dispatch(string $action, mixed ...$args): void` |
| `filter` | — | filter register | `Hook::filter(string $name, callable $cb, int $priority = 10): string` |
| `apply` | — | filter run | `Hook::apply(string $name, mixed $value, mixed ...$args): mixed` |
| `dispatchAsync` | — | action fire (task worker) | `Hook::dispatchAsync(string $action, mixed ...$args): int` |
| `remove` | — | unregister | `Hook::remove(string $name, string $id): bool` |
| `removeAll` | — | unregister | `Hook::removeAll(string $name): int` |
| `has` | — | introspection | `Hook::has(string $name): bool` |
| `listeners` | — | introspection | `Hook::listeners(string $name): array` |
| `names` | — | introspection | `Hook::names(?string $prefix = null): array` |
| `dispatching` | — | introspection | `Hook::dispatching(): ?string` |

> `listen`/`on` and `dispatch`/`do` are true aliases — identical behavior, provided so plugin authors can choose the phrasing that reads best (`Hook::on(...)` / `Hook::do(...)`). `filter` and `apply` intentionally have no alias to keep the value-pipeline semantics unambiguous.

---

## Registering Listeners

### Actions with `listen` / `on`

`Hook::listen()` registers a callback against an action name and returns an opaque **listener id** (used later for removal). Callback arguments are the variadic `...$args` passed to `Hook::dispatch()`. Return values are discarded.

```php
use Goco\SDK\Hook;

// Fire-and-forget: warm a cache when a page is published.
Hook::listen('content.published', function (array $doc): void {
    if (($doc['type'] ?? null) === 'page') {
        Cache::forget("page:{$doc['website_id']}:{$doc['slug']}");
    }
});

// `on` is an exact alias of `listen`.
Hook::on('user.login', fn (array $user) => Audit::record('login', $user['_id']));
```

### Filters with `filter`

`Hook::filter()` registers a transformer. The callback **must** accept the current value as its first parameter and **return** the next value. Additional context arguments passed to `Hook::apply()` arrive after the value but must not be returned.

```php
use Goco\SDK\Hook;

// Append a site suffix to every rendered page title.
Hook::filter('page.title', function (string $title, array $page): string {
    return $title === '' ? 'Home' : "{$title} — Acme Corp";
});

// Inject a security header into every response.
Hook::filter('response.headers', function (array $headers): array {
    $headers['X-Content-Type-Options'] = 'nosniff';
    return $headers;
}, priority: 20);
```

> **Warning** — A filter that forgets to `return` collapses the value to `null`, silently breaking every downstream listener and the caller. Always return; when in doubt, `return $value;` unchanged.

### Priorities and ordering

Listeners for a given name run in **ascending priority order** (lower runs first); the default is `10`. Listeners sharing a priority run in **registration order** (stable). For filters, ordering defines the transformation chain — an early listener's output is a later listener's input.

```php
Hook::filter('menu.items', $addDashboard, priority: 5);   // runs first
Hook::filter('menu.items', $reorderByWeight, priority: 10); // then this
Hook::filter('menu.items', $hideForGuests, priority: 90);  // runs last
```

| Priority band | Convention |
| --- | --- |
| `0`–`9` | Early / structural changes, defaults, must-run-first setup |
| `10` | Standard listeners (default) |
| `11`–`50` | Ordinary customization |
| `51`–`99` | Late overrides that must win |
| `100`+ | Final cleanup, logging, "always last" observers |

---

## Dispatching

### Actions with `dispatch` / `do`

```php
use Goco\SDK\Hook;

// Core fires an action after a document is published.
Hook::dispatch('content.published', $document, $context);

// `do` is an exact alias.
Hook::do('plugin.activated', $slug);
```

`Hook::dispatch()` runs every registered listener synchronously, in priority order, on the current coroutine, and returns `void`. Exceptions thrown by an individual listener are caught, logged (to `/tmp/zealphp/`), and isolated — one failing listener never aborts the dispatch or the request. See [Security Model](../security/security-model.md) for the failure-containment guarantees.

### Filters with `apply`

```php
use Goco\SDK\Hook;

$title = Hook::apply('page.title', $rawTitle, $page);            // subject.noun
$criteria = Hook::apply('query.criteria', $criteria, $ctx);      // mutate a query
$html = Hook::apply('widget.output', $renderedHtml, $type, $props);
```

`Hook::apply()` threads `$value` through each filter and returns the final value. If no filters are registered, the original `$value` is returned unchanged — filters are always safe to `apply` even when nothing listens.

> **Tip** — Give every `apply()` call a sensible default value so the feature works with zero plugins installed. Filters *refine* a working value; they should never be *required* to produce one.

---

## Asynchronous Events

`Hook::dispatchAsync()` offloads an action to an **OpenSwoole task worker** instead of running listeners inline. It returns immediately with a task id, keeping the request coroutine free. Use it for slow, non-blocking-optional side effects: sending mail via [Mailpit](../deployment/docker.md)/SMTP, webhooks, search reindexing, thumbnail generation, analytics.

```php
use Goco\SDK\Hook;

// Return the response fast; do the heavy work off-request.
Hook::dispatchAsync('content.published', $document);
// -> queues a task; listeners run in a task worker.
```

Semantics that differ from synchronous `dispatch`:

- **Serialization** — arguments cross a process boundary and must be serializable. Pass plain arrays / ids, not live coroutine handles, open sockets, or `\OpenSwoole\Coroutine\Channel` objects. Prefer passing a document `_id` and re-loading inside the listener.
- **No return path** — the caller cannot observe listener results (it is fire-and-forget by construction).
- **Ordering** — priority ordering still holds *within* the task, but the task itself runs after the current request yields.
- **Backpressure** — tasks are bounded by `task_worker_num`; when saturated, dispatch falls back to the Redis-backed [queue](../architecture/caching-and-queue.md) so events are never dropped.

Task workers are configured in the ZealPHP server options; register worker-side timers in `App::onWorkerStart()` when a listener needs periodic flushing.

```php
use ZealPHP\App;

App::onWorkerStart(function ($server, $wid) {
    // Flush a batched analytics buffer every 30s on task workers.
    App::tick(30000, fn () => Analytics::flush());
});
```

> **Note** — Filters have no async form. A filter must return a value to its caller synchronously; there is no `applyAsync`.

---

## Removing Listeners

Registration returns an id; keep it to remove precisely. This matters for plugin deactivation, where you must undo everything you registered.

```php
use Goco\SDK\Hook;

$id = Hook::listen('request.received', $tracer);

// Later — remove exactly that listener.
Hook::remove('request.received', $id);

// Remove every listener on a name (use sparingly; affects other plugins).
$count = Hook::removeAll('acme-crm.contact.synced');
```

Anonymous closures cannot be matched by reference, which is why `remove()` takes the returned id rather than the callable. Plugins should track their ids on the plugin object and unregister them in the deactivation lifecycle hook:

```php
final class AcmeCrmPlugin
{
    /** @var list<array{string,string}> */
    private array $hooks = [];

    public function boot(): void
    {
        $this->hooks[] = ['content.published', Hook::listen('content.published', [$this, 'syncContact'])];
        $this->hooks[] = ['user.login',        Hook::on('user.login',            [$this, 'touchLastSeen'])];
    }

    public function deactivate(): void
    {
        foreach ($this->hooks as [$name, $id]) {
            Hook::remove($name, $id);
        }
        $this->hooks = [];
    }
}
```

See [Plugin SDK](plugin-sdk.md) for how `boot()`/`deactivate()` fit the plugin lifecycle.

---

## Introspection

| Call | Returns | Use |
| --- | --- | --- |
| `Hook::has('page.title')` | `bool` | Guard expensive `apply()` setup when nobody listens |
| `Hook::listeners('page.title')` | `array` of `{id, priority, source}` | Debug ordering, build an admin "hooks" panel |
| `Hook::names('acme-crm.')` | `array` of registered names with that prefix | Discover a plugin's surface |
| `Hook::dispatching()` | `?string` current hook name or `null` | Detect / prevent re-entrancy |

```php
use Goco\SDK\Hook;

if (Hook::has('report.export.rows')) {
    $rows = Hook::apply('report.export.rows', $rows, $ctx);
}

foreach (Hook::listeners('menu.items') as $l) {
    logger()->debug("menu.items <- {$l['source']} @ {$l['priority']}");
}
```

`Hook::listeners()` powers the developer tools panel in the admin app, letting you see, per hook, which plugin registered what and at which priority — invaluable when two plugins fight over `page.title`.

---

## Naming Rules

Hook names are **lowercase, dot-separated, ASCII**. The grammar encodes intent so a name alone tells you whether to `dispatch` or `apply`.

| Kind | Pattern | Meaning | Examples |
| --- | --- | --- | --- |
| **Action** | `subject.verb[.tense]` | An event occurred | `core.boot`, `request.received`, `content.publishing`, `content.published`, `widget.render.before`, `widget.render.after`, `plugin.activated`, `user.login` |
| **Filter** | `subject.noun` | A value to transform | `page.title`, `widget.output`, `menu.items`, `query.criteria`, `response.headers` |

Rules:

1. **Tense signals timing.** Present-progressive (`content.publishing`) fires *before* the change and may still be blocked by a listener throwing; past tense (`content.published`) fires *after* it commits.
2. **`before`/`after` pairs** bracket a specific operation (`widget.render.before` / `widget.render.after`).
3. **Namespace plugin hooks by slug.** Any hook a plugin *emits* must be prefixed with the plugin slug: `acme-crm.contact.synced`, `acme-crm.export.rows`. This prevents collisions and makes `Hook::names('acme-crm.')` a complete discovery tool. Core hooks are reserved and never take a plugin prefix.
4. **Listen to anyone; emit under your slug.** A plugin may `listen`/`filter` on core hooks freely, but should only `dispatch`/`apply` its own namespaced names.

```php
// GOOD — plugin emits under its slug, listens on core freely.
Hook::dispatch('acme-crm.contact.synced', $contact);
Hook::filter('page.title', $addBrand);

// BAD — plugin emitting a bare/core-looking name (collision risk).
Hook::dispatch('contact.synced', $contact);   // ambiguous
Hook::dispatch('content.published', $doc);     // impersonates core
```

---

## Core Hook Catalog (compact)

A representative subset. The authoritative, exhaustive list lives in [Event & Hook System](../architecture/event-hook-system.md).

| Name | Kind | Fired / applied | Payload / value |
| --- | --- | --- | --- |
| `core.boot` | action | Once, after the app initializes | `App $app` |
| `request.received` | action | Start of every request | `Request $request` |
| `page.rendering` | action | Before a page renders | `array $page, Context $ctx` |
| `page.rendered` | action | After a page's HTML is built | `string $html, array $page` |
| `content.publishing` | action | Before publish commits (blockable) | `array $doc` |
| `content.published` | action | After publish commits | `array $doc` |
| `widget.render.before` | action | Before a widget renders | `string $type, array $props` |
| `widget.render.after` | action | After a widget renders | `string $type, string $html` |
| `plugin.activated` | action | After a plugin is enabled | `string $slug` |
| `user.login` | action | After successful authentication | `array $user` |
| `page.title` | filter | Page `<title>` composition | `string` → `string` |
| `widget.output` | filter | Final widget HTML | `string` → `string` |
| `menu.items` | filter | Navigation menu tree | `array` → `array` |
| `query.criteria` | filter | Repository query criteria | `array` → `array` |
| `response.headers` | filter | Outgoing HTTP headers | `array` → `array` |

---

## Best Practices

**Keep hot hooks cheap.** `request.received`, `widget.render.before`, and `page.title` run on every request or every widget. Do no blocking I/O there; if you must fetch, use a coroutine-friendly cache read against [Redis](../architecture/caching-and-queue.md) or defer with `dispatchAsync`.

```php
// BAD — synchronous HTTP call on the hot path serializes the request.
Hook::on('request.received', fn ($req) => file_get_contents('https://geo.example/lookup'));

// GOOD — offload; enrich later.
Hook::on('request.received', fn ($req) => Hook::dispatchAsync('acme-geo.enrich', $req->ip()));
```

**Make listeners idempotent.** Async retries and at-least-once task delivery mean a listener may run more than once for the same event. Key side effects so a repeat is a no-op (`upsert` by `_id`, `SETNX` a Redis lock, dedupe on an idempotency key).

```php
Hook::on('content.published', function (array $doc) {
    $key = "reindex:{$doc['_id']}:{$doc['version']}";
    if (Redis::set($key, 1, ['NX', 'EX' => 3600])) {
        Search::index($doc);   // runs once per (doc, version)
    }
});
```

**Filters are pure transforms.** A filter should compute a return value from its inputs and avoid side effects (no DB writes, no dispatching other actions from inside `apply`). Reserve mutation for actions.

**Never assume order across plugins.** Pin the priority you actually need instead of relying on registration timing between independently installed plugins.

**Do not re-enter the same hook.** Dispatching `content.published` from within a `content.published` listener risks infinite recursion; use `Hook::dispatching()` to guard, or dispatch a distinct namespaced follow-up event.

**Namespace and document your emitted hooks.** Anything you `dispatch`/`apply` under your slug is public API for downstream plugins — treat renames as breaking changes under [semantic versioning](../roadmap.md).

---

## Testing Hooks

Hooks are ordinary callables and test cleanly. Reset the registry between tests, register, exercise, and assert on captured state or returned values.

```php
use Goco\SDK\Hook;
use PHPUnit\Framework\TestCase;

final class BrandTitleFilterTest extends TestCase
{
    protected function setUp(): void
    {
        Hook::removeAll('page.title'); // isolate: clear inherited listeners
    }

    public function test_filter_appends_brand(): void
    {
        Hook::filter('page.title', fn (string $t) => "{$t} — Acme");

        $out = Hook::apply('page.title', 'Pricing', ['slug' => 'pricing']);

        $this->assertSame('Pricing — Acme', $out);
    }

    public function test_action_records_side_effect(): void
    {
        $seen = [];
        Hook::on('content.published', function (array $doc) use (&$seen) {
            $seen[] = $doc['_id'];
        });

        Hook::dispatch('content.published', ['_id' => 'p_1', 'type' => 'page']);

        $this->assertSame(['p_1'], $seen);
    }

    public function test_priority_orders_filters(): void
    {
        Hook::filter('menu.items', fn (array $m) => [...$m, 'b'], priority: 20);
        Hook::filter('menu.items', fn (array $m) => [...$m, 'a'], priority: 10);

        $this->assertSame(['a', 'b'], Hook::apply('menu.items', []));
    }
}
```

For async listeners, test the **listener body directly** as a plain function rather than routing through task workers — assert the effect given a payload. Reserve full `dispatchAsync` round-trips for integration tests that spin up the OpenSwoole server. See [Testing Strategy](../community/testing-strategy.md) for the project's fixtures, the `goco test` runner, and coroutine-aware assertions.

> **Tip** — Call `Hook::removeAll($name)` (or reset the registry in a shared test bootstrap) in `setUp()` so listeners from other suites never leak into your assertions.

---

## Related

- [Event & Hook System](../architecture/event-hook-system.md) — engine internals and the full hook catalog
- [Plugin SDK](plugin-sdk.md) — lifecycle where you register and remove hooks
- [Widget SDK](widget-sdk.md) — hooks around widget rendering
- [Theme SDK](theme-sdk.md) — hooks around layouts, regions, and assets
- [CLI SDK](cli.md) — generators and lifecycle commands
- [Plugin Engine](../core/plugin-engine.md) — how plugins are loaded and booted
- [Caching, Queue & Realtime (Redis)](../architecture/caching-and-queue.md) — task workers and the async queue backend
- [Security Model](../security/security-model.md) — listener isolation and failure containment
- [Testing Strategy](../community/testing-strategy.md) — testing conventions
- [Documentation Index](../README.md)
