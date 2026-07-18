# Template Guide

> A complete, end-to-end tutorial: build, wire, validate, and package a GOCO CMS **template** for a CSS framework (Bootstrap 5) — from empty folder to signed, marketplace-ready artifact that any theme can consume.

**Stability:** `beta` · **Package:** `gococms/template-engine` · **Namespace:** `Goco\Template` · **CLI:** `goco template:*` · **Facades:** `Goco\SDK\Theme`, `Goco\SDK\Widget`, `Goco\SDK\Hook`

This guide walks you through authoring a real template package called **`horizon`** targeting **Bootstrap 5** (`bootstrap` adapter). By the end you will have layouts, reusable components, per-template widget renderers, fingerprinted assets, localized strings, hook wiring, a valid `template.json`, a generated signed `manifest.json`, and a `.gocotpl` bundle you can publish to the [Plugin Marketplace](../marketplace/overview.md).

If you have not yet, read the conceptual reference first: the [Template Engine](../core/template-engine.md) explains *what* a template package is and how it renders; the [Theme Engine](../core/theme-engine.md) explains how a theme *consumes* templates. This guide is the *how-to*.

> **Note**
> A template **package** (this guide) is a distributable bundle of layouts, components, assets, and metadata. A template **file** is a single `.php` view rendered by `App::render`. A package *contains* many files. Do not confuse the two.

---

## Prerequisites

Before you start, confirm the toolchain and a working GOCO checkout.

| Requirement | Version | Check |
| --- | --- | --- |
| PHP | 8.4+ | `php -v` |
| OpenSwoole | 22.1+ | `php --ri openswoole` |
| GOCO CLI (`goco`) | matches your core | `goco --version` |
| Node (asset build, optional) | 20+ | `node -v` |
| A running GOCO stack | Docker Compose | `docker compose ps` |

```bash
# From the monorepo root — bring the dev stack up (gococms, mongodb, redis, traefik, minio, meilisearch, mailpit)
docker compose up -d
docker compose ps            # all services should report healthy
goco status                  # ZealPHP runtime status (logs live in /tmp/zealphp/)
```

> **Tip**
> Templates live under the monorepo `templates/<slug>/` directory (see [Project Structure](../getting-started/project-structure.md)). During development, `goco template:link` symlinks your working copy into the running runtime so edits are picked up on the next request without reinstalling.

---

## Step 1 — Scaffold the package

Generate the folder contract with the CLI, then inspect it. The scaffolder writes a compliant skeleton so you never start from a blank directory.

```bash
goco template:new horizon \
  --name "Horizon" \
  --framework bootstrap \
  --framework-version 5.3 \
  --author "Acme Studio <dev@acme.example>" \
  --license MIT
```

This creates `templates/horizon/` with the exact on-disk contract the engine expects:

```text
templates/horizon/
├── template.json            # descriptor — you author this
├── manifest.json            # build artifact — generated, never hand-edit
├── README.md                # usage, screenshots, changelog
├── layouts/                 # full-page skeletons that expose named regions
│   ├── default.php
│   ├── landing.php
│   └── blog.php
├── widgets/                 # per-template widget renderer views
│   ├── hero-banner.php
│   └── post-list.php
├── components/              # reusable partials + prop schemas
│   ├── navbar.php
│   ├── navbar.schema.json
│   ├── hero.php
│   ├── hero.schema.json
│   ├── card.php
│   ├── card.schema.json
│   └── pagination.php
├── assets/                  # source + built front-end assets
│   ├── css/
│   │   ├── app.css
│   │   └── critical.css
│   ├── js/
│   │   └── app.js
│   ├── images/
│   │   └── placeholder.svg
│   └── fonts/
│       └── Inter-var.woff2
├── translations/            # i18n message catalogs (one JSON per locale)
│   ├── en.json
│   ├── es.json
│   └── ta.json
└── hooks/                   # Hook::listen / Hook::filter registrations
    ├── register.php
    └── filters.php
```

> **Warning**
> `manifest.json` and every fingerprinted `assets/**/*.<hash>.*` path are produced by `goco template:build`. Editing them by hand desynchronizes integrity hashes, and the marketplace signature check will reject the package on install.

Link it into the running runtime so you can preview as you build:

```bash
goco template:link horizon         # symlink into the active runtime
goco template:list                 # 'horizon' should appear as (linked, dev)
```

---

## Step 2 — Author `template.json`

`template.json` is the human-authored **descriptor**: identity, engine compatibility, the target framework/adapter, the layout → region map, the component index, asset entry points, and locales. This is the single source of truth the engine reads at install and render time.

```json
{
  "slug": "horizon",
  "name": "Horizon",
  "version": "1.0.0",
  "description": "A marketing + blog template for Bootstrap 5: navbar, hero, feature cards, and a paginated post list.",
  "author": { "name": "Acme Studio", "email": "dev@acme.example", "url": "https://acme.example" },
  "license": "MIT",
  "keywords": ["bootstrap", "marketing", "blog", "responsive"],
  "engine": { "goco": ">=0.9 <1.0", "php": ">=8.4", "zealphp": ">=1.0" },
  "framework": {
    "adapter": "bootstrap",
    "version": "5.3",
    "options": { "theme": "auto", "container": "container", "gutters": "g-4" }
  },
  "layouts": {
    "default": {
      "title": "Default",
      "file": "layouts/default.php",
      "regions": ["navbar", "main", "footer"]
    },
    "landing": {
      "title": "Landing Page",
      "file": "layouts/landing.php",
      "regions": ["navbar", "hero", "features", "cta", "footer"]
    },
    "blog": {
      "title": "Blog Index",
      "file": "layouts/blog.php",
      "regions": ["navbar", "posts", "sidebar", "footer"]
    }
  },
  "regions": {
    "navbar":   { "label": "Navigation Bar", "max": 1 },
    "hero":     { "label": "Hero",           "max": 1 },
    "features": { "label": "Feature Grid",   "max": 12 },
    "cta":      { "label": "Call to Action", "max": 1 },
    "main":     { "label": "Main Content",   "max": null },
    "posts":    { "label": "Post List",      "max": null },
    "sidebar":  { "label": "Sidebar",        "max": 8 },
    "footer":   { "label": "Footer",         "max": 1 }
  },
  "components": {
    "navbar":     { "file": "components/navbar.php",     "schema": "components/navbar.schema.json" },
    "hero":       { "file": "components/hero.php",       "schema": "components/hero.schema.json" },
    "card":       { "file": "components/card.php",       "schema": "components/card.schema.json" },
    "pagination": { "file": "components/pagination.php", "schema": null }
  },
  "widgets": {
    "hero-banner": { "file": "widgets/hero-banner.php" },
    "post-list":   { "file": "widgets/post-list.php" }
  },
  "assets": {
    "css": ["assets/css/app.css"],
    "critical": ["assets/css/critical.css"],
    "js": ["assets/js/app.js"],
    "fonts": ["assets/fonts/Inter-var.woff2"],
    "vendor": {
      "bootstrap": { "css": "cdn:bootstrap@5.3", "js": "cdn:bootstrap@5.3/bundle" }
    }
  },
  "i18n": { "default": "en", "locales": ["en", "es", "ta"], "rtl": ["ar", "he"] },
  "capabilities": ["themes.manage", "widgets.manage"],
  "hooks": ["hooks/register.php", "hooks/filters.php"]
}
```

Field-by-field, the load-bearing keys:

| Key | Purpose |
| --- | --- |
| `framework.adapter` | Selects the `bootstrap` `FrameworkAdapter`; every layout primitive (grid/column/spacing/visibility) resolves through it. |
| `framework.options` | Adapter-specific defaults (Bootstrap container class, gutter token, color mode) merged into render context. |
| `layouts.<id>.regions` | The named regions a theme maps widgets into; must match placeholders in the layout view. |
| `regions.<id>.max` | Capacity hint for the visual [Page Builder](../core/page-builder.md); `null` = unbounded. |
| `components` | Index of reusable partials; `schema` is a JSON-Schema for typed props (validated at render in dev). |
| `widgets` | Per-template renderer views a registered `Widget` type renders *through* for this template. |
| `i18n.locales` | Locales for which a `translations/<locale>.json` catalog must exist. |
| `capabilities` | RBAC capabilities the install requires (see [Permission System](../architecture/permission-system.md)). |

---

## Step 3 — Build reusable components

Components are parameterized, escaped presentational partials rendered with `App::render` (full string) or `App::fragment` (htmx region). Each component pairs a `.php` view with an optional `.schema.json` that validates its props in development.

### 3.1 The Bootstrap adapter contract

Do not hand-write grid classes. Ask the adapter — the same code then re-emits correct markup if the template's framework changes. In a view, the resolved adapter is available as `$adapter`, and the translator as `$t`.

```php
// $adapter is a Goco\Template\Adapter\FrameworkAdapter (bootstrap)
$adapter->grid(12);                     // 'row'
$adapter->column(4, 12, ['md' => 6]);   // 'col-12 col-md-6 col-lg-4'
$adapter->spacing('gap-lg');            // 'g-4'
$adapter->visibility(['md' => false]);  // 'd-none d-md-block'
$adapter->stateClass('active');         // 'active'
```

### 3.2 `components/card.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Card",
  "type": "object",
  "required": ["title"],
  "additionalProperties": false,
  "properties": {
    "title":   { "type": "string", "maxLength": 120 },
    "body":    { "type": "string" },
    "href":    { "type": "string", "format": "uri-reference" },
    "image":   { "type": "string" },
    "cta":     { "type": "string", "default": "card.read_more" },
    "variant": { "type": "string", "enum": ["default", "shadow", "border"], "default": "shadow" }
  }
}
```

### 3.3 `components/card.php`

```php
<?php
/**
 * Reusable Bootstrap 5 card.
 * Props (validated against card.schema.json in dev):
 *   string  $title    Card heading
 *   ?string $body     Body text
 *   ?string $href     Link target for the CTA
 *   ?string $image    Image URL (asset-resolved)
 *   string  $cta      Translation key for the CTA label
 *   string  $variant  default|shadow|border
 *
 * Injected by the engine: $adapter (FrameworkAdapter), $t (translator), $asset (asset resolver).
 */
$variant = $variant ?? 'shadow';
$cardClass = 'card h-100 ' . match ($variant) {
    'border' => 'border',
    'default' => '',
    default  => 'shadow-sm',
};
?>
<div class="<?= htmlspecialchars($cardClass) ?>">
  <?php if (!empty($image)): ?>
    <img src="<?= htmlspecialchars($asset($image)) ?>" class="card-img-top" alt="" loading="lazy">
  <?php endif; ?>
  <div class="card-body d-flex flex-column">
    <h3 class="card-title h5"><?= htmlspecialchars($title) ?></h3>
    <?php if (!empty($body)): ?>
      <p class="card-text flex-grow-1"><?= htmlspecialchars($body) ?></p>
    <?php endif; ?>
    <?php if (!empty($href)): ?>
      <a href="<?= htmlspecialchars($href) ?>" class="btn btn-primary mt-2 align-self-start">
        <?= htmlspecialchars($t($cta ?? 'card.read_more')) ?>
      </a>
    <?php endif; ?>
  </div>
</div>
```

### 3.4 `components/navbar.php`

The navbar consumes `menu.items` (filterable — see Step 6) and renders a Bootstrap collapse.

```php
<?php
/** Props: string $brand, array $items [{label,url,active}], ?string $logo */
$items = \Goco\SDK\Hook::apply('menu.items', $items ?? [], 'primary');
?>
<nav class="navbar navbar-expand-lg bg-body-tertiary sticky-top">
  <div class="<?= htmlspecialchars($adapter->option('container', 'container')) ?>">
    <a class="navbar-brand d-flex align-items-center gap-2" href="/">
      <?php if (!empty($logo)): ?><img src="<?= htmlspecialchars($asset($logo)) ?>" height="28" alt=""><?php endif; ?>
      <span><?= htmlspecialchars($brand ?? $t('site.name')) ?></span>
    </a>
    <button class="navbar-toggler" type="button" data-bs-toggle="collapse"
            data-bs-target="#horizonNav" aria-controls="horizonNav"
            aria-expanded="false" aria-label="<?= htmlspecialchars($t('navbar.toggle')) ?>">
      <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="horizonNav">
      <ul class="navbar-nav ms-auto mb-2 mb-lg-0">
        <?php foreach ($items as $item): ?>
          <li class="nav-item">
            <a class="nav-link <?= !empty($item['active']) ? $adapter->stateClass('active') : '' ?>"
               <?= !empty($item['active']) ? 'aria-current="page"' : '' ?>
               href="<?= htmlspecialchars($item['url']) ?>"><?= htmlspecialchars($item['label']) ?></a>
          </li>
        <?php endforeach; ?>
      </ul>
    </div>
  </div>
</nav>
```

### 3.5 `components/hero.php` and `components/pagination.php`

```php
<?php /* components/hero.php — Props: string $heading, ?string $subheading, ?string $ctaText, ?string $ctaUrl */ ?>
<section class="py-5 text-center bg-body-tertiary">
  <div class="<?= htmlspecialchars($adapter->option('container', 'container')) ?> py-lg-5">
    <h1 class="display-4 fw-bold"><?= htmlspecialchars($heading) ?></h1>
    <?php if (!empty($subheading)): ?>
      <p class="lead col-lg-8 mx-auto"><?= htmlspecialchars($subheading) ?></p>
    <?php endif; ?>
    <?php if (!empty($ctaUrl)): ?>
      <a class="btn btn-lg btn-primary mt-3" href="<?= htmlspecialchars($ctaUrl) ?>">
        <?= htmlspecialchars($t($ctaText ?? 'hero.cta')) ?>
      </a>
    <?php endif; ?>
  </div>
</section>
```

```php
<?php
/** components/pagination.php — Props: int $page, int $pages, string $base */
if (($pages ?? 1) < 2) return;
?>
<nav aria-label="<?= htmlspecialchars($t('pagination.label')) ?>">
  <ul class="pagination justify-content-center">
    <li class="page-item <?= $page <= 1 ? 'disabled' : '' ?>">
      <a class="page-link" href="<?= htmlspecialchars($base . '?page=' . max(1, $page - 1)) ?>"><?= htmlspecialchars($t('pagination.prev')) ?></a>
    </li>
    <?php for ($i = 1; $i <= $pages; $i++): ?>
      <li class="page-item <?= $i === $page ? $adapter->stateClass('active') : '' ?>">
        <a class="page-link" href="<?= htmlspecialchars($base . '?page=' . $i) ?>"><?= $i ?></a>
      </li>
    <?php endfor; ?>
    <li class="page-item <?= $page >= $pages ? 'disabled' : '' ?>">
      <a class="page-link" href="<?= htmlspecialchars($base . '?page=' . min($pages, $page + 1)) ?>"><?= htmlspecialchars($t('pagination.next')) ?></a>
    </li>
  </ul>
</nav>
```

> **Tip**
> Every prop is HTML-escaped at output. The `$t()` translator returns escaped strings by default. Only reach for the explicit `raw()` helper when you have already sanitized markup — see the [Security Model](../security/security-model.md).

---

## Step 4 — Build layouts with named regions

A **layout** is a full-page skeleton that declares the regions listed in `template.json`. Regions are filled by the engine with the widgets a theme/page assigns to them. Use `App::include()` to compose components inside a layout, and `App::fragment()` to isolate an htmx-updatable region.

### 4.1 `layouts/default.php`

```php
<?php
/**
 * Default layout — regions: navbar, main, footer.
 * The engine injects: $regions (rendered HTML per region), $adapter, $t, $asset, $meta.
 */
$container = $adapter->option('container', 'container');
?>
<!doctype html>
<html lang="<?= htmlspecialchars($meta['locale'] ?? 'en') ?>"
      dir="<?= htmlspecialchars($meta['dir'] ?? 'ltr') ?>"
      data-bs-theme="<?= htmlspecialchars($adapter->option('theme', 'auto')) ?>">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title><?= htmlspecialchars(\Goco\SDK\Hook::apply('page.title', $meta['title'] ?? $t('site.name'))) ?></title>
  <?= $asset->styles('critical') /* inlined critical CSS */ ?>
  <?= $asset->styles('app') ?>
</head>
<body>
  <?= $regions['navbar'] ?? App::include('/templates/horizon/components/navbar.php', ['brand' => $t('site.name')]) ?>
  <main class="<?= htmlspecialchars($container) ?> py-4">
    <?= $regions['main'] ?? '' ?>
  </main>
  <?= $regions['footer'] ?? '' ?>
  <?= $asset->scripts('app') ?>
</body>
</html>
```

### 4.2 `layouts/landing.php`

The landing layout renders hero + a responsive feature grid through the adapter. Note how `$adapter->grid()` / `$adapter->column()` produce Bootstrap classes with zero literal grid strings.

```php
<?php /* regions: navbar, hero, features, cta, footer */
$container = $adapter->option('container', 'container');
$gutters   = $adapter->spacing('gap-lg');
?>
<!doctype html>
<html lang="<?= htmlspecialchars($meta['locale'] ?? 'en') ?>" data-bs-theme="<?= htmlspecialchars($adapter->option('theme','auto')) ?>">
<head>
  <meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1">
  <title><?= htmlspecialchars(\Goco\SDK\Hook::apply('page.title', $meta['title'] ?? $t('site.name'))) ?></title>
  <?= $asset->styles('critical') ?><?= $asset->styles('app') ?>
</head>
<body>
  <?= $regions['navbar'] ?? '' ?>
  <?= $regions['hero'] ?? '' ?>

  <section class="<?= htmlspecialchars($container) ?> py-5">
    <div class="<?= htmlspecialchars($adapter->grid(12) . ' ' . $gutters) ?>">
      <?php foreach (($features ?? []) as $feature): ?>
        <div class="<?= htmlspecialchars($adapter->column(4, 12, ['sm' => 6])) ?>">
          <?= App::include('/templates/horizon/components/card.php', $feature) ?>
        </div>
      <?php endforeach; ?>
    </div>
  </section>

  <?= $regions['cta'] ?? '' ?>
  <?= $regions['footer'] ?? '' ?>
  <?= $asset->scripts('app') ?>
</body>
</html>
```

### 4.3 `layouts/blog.php` with an htmx fragment region

The `posts` region re-renders on its own via `App::fragment()`, so pagination swaps only the list — not the whole page.

```php
<?php /* regions: navbar, posts, sidebar, footer */
$container = $adapter->option('container', 'container');
?>
<!doctype html>
<html lang="<?= htmlspecialchars($meta['locale'] ?? 'en') ?>" data-bs-theme="auto">
<head>
  <meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1">
  <title><?= htmlspecialchars($meta['title'] ?? $t('blog.title')) ?></title>
  <?= $asset->styles('app') ?>
</head>
<body>
  <?= $regions['navbar'] ?? '' ?>
  <div class="<?= htmlspecialchars($container) ?> py-4">
    <div class="<?= htmlspecialchars($adapter->grid(12)) ?>">
      <div class="<?= htmlspecialchars($adapter->column(8)) ?>">
        <?= App::fragment('posts', '/templates/horizon/widgets/post-list.php', [
              'posts' => $posts ?? [], 'page' => $page ?? 1, 'pages' => $pages ?? 1, 'base' => $meta['path'] ?? '/blog',
            ]) ?>
      </div>
      <aside class="<?= htmlspecialchars($adapter->column(4)) ?>">
        <?= $regions['sidebar'] ?? '' ?>
      </aside>
    </div>
  </div>
  <?= $regions['footer'] ?? '' ?>
  <?= $asset->scripts('app') ?>
</body>
</html>
```

---

## Step 5 — Per-template widget renderers

A GOCO widget type (registered once, framework-agnostic) renders *through* the active template's renderer view. That means the same `hero-banner` widget emits Bootstrap markup under this template and Tailwind markup under another — only the renderer file differs.

### 5.1 `widgets/hero-banner.php`

```php
<?php
/**
 * Renderer for the core 'hero-banner' widget under the Horizon (Bootstrap 5) template.
 * $props are the widget's validated properties; $ctx is the render Context.
 */
echo App::include('/templates/horizon/components/hero.php', [
  'heading'    => $props['heading']    ?? $t('hero.default_heading'),
  'subheading' => $props['subheading'] ?? null,
  'ctaText'    => $props['cta_text']   ?? 'hero.cta',
  'ctaUrl'     => $props['cta_url']    ?? null,
]);
```

### 5.2 `widgets/post-list.php` (streamable)

Returning a generator lets the engine stream rows progressively; `co::sleep()` yields the coroutine between chunks so a slow feed never blocks the worker.

```php
<?php
/** Renderer for 'post-list'. Streams cards, then the pagination control. */
use OpenSwoole\Coroutine as co;

return (function () use ($posts, $page, $pages, $base, $adapter, $t, $asset) {
    yield '<div class="' . htmlspecialchars($adapter->grid(12) . ' ' . $adapter->spacing('gap-lg')) . '" id="post-list">';
    foreach (($posts ?? []) as $post) {
        yield '<div class="' . htmlspecialchars($adapter->column(6)) . '">'
            . App::include('/templates/horizon/components/card.php', [
                'title' => $post['title'], 'body' => $post['excerpt'] ?? '',
                'href' => $post['url'], 'image' => $post['cover'] ?? null, 'cta' => 'card.read_more',
              ])
            . '</div>';
        co::sleep(0.001); // cooperative yield between rows
    }
    yield '</div>';
    yield App::include('/templates/horizon/components/pagination.php', [
        'page' => $page, 'pages' => $pages, 'base' => $base,
    ]);
})();
```

> **Note**
> How a widget *type* is defined (its property schema, defaults, preview) is out of scope here — see the [Widget Guide](widget-guide.md) and [Widget SDK](../sdk/widget-sdk.md). This step only provides the *Bootstrap rendering* for widgets your template supports.

---

## Step 6 — Wire hooks

`hooks/*.php` files run when the template boots. Use them to register listeners and filters, all namespaced to the template slug so they never collide with other packages. Follow the canonical hook naming: actions are `subject.verb[.tense]`, filters are `subject.noun`.

### 6.1 `hooks/register.php`

```php
<?php
use Goco\SDK\Hook;
use Goco\SDK\Widget;

// Bind this template's Bootstrap renderers to core widget types.
Hook::listen('template.horizon.boot', function () {
    Widget::register('hero-banner', ['renderer' => 'templates/horizon/widgets/hero-banner.php']);
    Widget::register('post-list',   ['renderer' => 'templates/horizon/widgets/post-list.php']);
}, priority: 10);

// Enqueue template assets once a page starts rendering.
Hook::listen('page.rendering', function ($ctx) {
    $ctx->assets()->enqueueCss('templates/horizon/assets/css/app.css');
    $ctx->assets()->enqueueJs('templates/horizon/assets/js/app.js', ['defer' => true]);
}, priority: 20);
```

### 6.2 `hooks/filters.php`

```php
<?php
use Goco\SDK\Hook;

// Append a template signature to the document title (subject.noun filter).
Hook::filter('page.title', function (string $title): string {
    return $title;
}, priority: 50);

// Normalize menu items so every entry has an 'active' flag for the navbar (subject.noun filter).
Hook::filter('menu.items', function (array $items, string $location): array {
    if ($location !== 'primary') return $items;
    $path = \Goco\Runtime\G::request()->path();
    foreach ($items as &$item) {
        $item['active'] = rtrim($item['url'], '/') === rtrim($path, '/');
    }
    return $items;
}, priority: 10);
```

For the full hook catalog, aliases (`on`/`do`), async dispatch, and priority semantics, see the [Hook SDK](../sdk/hook-sdk.md) and [Event & Hook System](../architecture/event-hook-system.md).

---

## Step 7 — Add translations

Every user-visible string flows through `$t('key')`, which the engine resolves against `translations/<locale>.json` with a fallback chain (`requested → i18n.default → key`). Ship one catalog per declared locale. Nested keys are addressed with dots (`card.read_more`).

### 7.1 `translations/en.json`

```json
{
  "site.name": "Horizon",
  "navbar.toggle": "Toggle navigation",
  "hero.default_heading": "Build faster with Horizon",
  "hero.cta": "Get started",
  "card.read_more": "Read more",
  "blog.title": "From the blog",
  "pagination.label": "Blog pagination",
  "pagination.prev": "Previous",
  "pagination.next": "Next"
}
```

### 7.2 `translations/es.json`

```json
{
  "site.name": "Horizon",
  "navbar.toggle": "Alternar navegación",
  "hero.default_heading": "Construye más rápido con Horizon",
  "hero.cta": "Comenzar",
  "card.read_more": "Leer más",
  "blog.title": "Desde el blog",
  "pagination.label": "Paginación del blog",
  "pagination.prev": "Anterior",
  "pagination.next": "Siguiente"
}
```

### 7.3 `translations/ta.json`

```json
{
  "site.name": "Horizon",
  "navbar.toggle": "வழிசெலுத்தலை மாற்று",
  "hero.default_heading": "Horizon உடன் வேகமாக உருவாக்குங்கள்",
  "hero.cta": "தொடங்குங்கள்",
  "card.read_more": "மேலும் படிக்க",
  "blog.title": "வலைப்பதிவிலிருந்து",
  "pagination.label": "வலைப்பதிவு பக்கமாக்கல்",
  "pagination.prev": "முந்தையது",
  "pagination.next": "அடுத்தது"
}
```

> **Warning**
> Missing keys fall back to `i18n.default`, then to the literal key. `goco template:validate` fails the build if a locale listed in `i18n.locales` is missing a key present in the default catalog, so keep catalogs in sync.

For RTL locales listed in `i18n.rtl` (e.g. `ar`, `he`), the engine sets `dir="rtl"` on the layout and the Bootstrap adapter emits logical (`ms-*`/`me-*`) spacing utilities automatically.

---

## Step 8 — Preview during development

With the template linked (Step 1), render a layout against sample data without a full site. The preview server mounts your working copy live.

```bash
# Render a single layout with fixture data and open a live preview at https://horizon.localhost
goco template:preview horizon --layout landing --fixtures examples/horizon/landing.json
```

Under the hood the runtime calls `App::render('/templates/horizon/layouts/landing.php', $data)` on each request (see the [Request Lifecycle](../architecture/request-lifecycle.md)). Because ZealPHP is coroutine-based, hot edits are visible on the next request — no restart. Tail logs if something misbehaves:

```bash
goco logs --follow          # ZealPHP runtime logs (also under /tmp/zealphp/)
```

> **Tip**
> Preview requests route through Traefik, so you get real HTTPS on `*.localhost` and the same header/middleware stack production uses. See [Traefik Reverse Proxy](../deployment/traefik.md).

---

## Step 9 — From template to usable theme

A template is *presentation building blocks*; a **theme** is the site's design identity that *consumes* a template and maps content into its regions. A template becomes usable once a theme references it. There are two paths.

### 9.1 Reference the template from a theme manifest

In a theme's `theme.json`, name the template package and, optionally, per-region defaults:

```json
{
  "slug": "acme-brand",
  "name": "Acme Brand",
  "version": "1.0.0",
  "template": { "slug": "horizon", "version": "^1.0" },
  "tokens": { "primary": "#0d6efd", "font-sans": "Inter, system-ui, sans-serif" },
  "layouts": { "home": "landing", "blog": "blog", "page": "default" },
  "regions": {
    "navbar": [{ "component": "navbar", "props": { "logo": "brand/logo.svg" } }],
    "footer": [{ "widget": "site-footer" }]
  }
}
```

Register and activate it with the SDK — the exact `Theme` facade signatures:

```php
use Goco\SDK\Theme;

Theme::register('acme-brand', $manifest);   // manifest read from theme.json
Theme::layouts('acme-brand');               // ['home' => 'landing', 'blog' => 'blog', 'page' => 'default']
Theme::regions('landing');                  // ['navbar','hero','features','cta','footer']
Theme::assets('acme-brand');                // AssetBundle: merged template + theme fingerprinted assets
```

See the [Theme Engine](../core/theme-engine.md) and [Theme SDK](../sdk/theme-sdk.md) for the full manifest schema, token cascade, and region-mapping rules.

### 9.2 Activate for a website

```bash
goco theme:install acme-brand
goco theme:activate acme-brand --website acme.example
```

At runtime the resolution chain is **Workspace → Website → Theme → Layout → Section → Container → Row → Column → Widget**: a request resolves the website's active theme, the theme names the `horizon` template and a layout id, the engine loads that layout, resolves the `bootstrap` adapter, fills regions by rendering assigned widgets through Horizon's renderers, and streams the HTML. See the [Rendering Pipeline](../architecture/rendering-pipeline.md).

---

## Step 10 — Validate

Run the validator before every build. It statically checks the descriptor, the folder contract, prop schemas, translation parity, and asset references.

```bash
goco template:validate horizon
```

What it enforces:

| Check | Rule |
| --- | --- |
| Descriptor | `template.json` matches the template JSON-Schema; `slug` is kebab-case; `version` is valid SemVer. |
| Engine range | `engine.goco` is a satisfiable range against the installed core. |
| Adapter | `framework.adapter` is a known adapter id (`bootstrap`, `tailwind`, `bulma`, `uikit`, `foundation`, `material`, `pure`, `custom`). |
| Layout ↔ region | Every region in `layouts.*.regions` exists in `regions`; every layout `file` exists. |
| Components | Each component `file` exists; where a `schema` is declared, it is valid JSON-Schema. |
| i18n parity | Every locale in `i18n.locales` has a catalog with the full default keyset. |
| Assets | Every path in `assets.*` exists; no missing font/image references. |
| Escaping lint | Flags raw echo of props not wrapped in `htmlspecialchars`/`$t`/`raw()`. |

A clean run prints `horizon@1.0.0 — 0 errors, 0 warnings`. Validation is also enforced in CI per the [Testing Strategy](../community/testing-strategy.md) and the [Coding Standards](../community/coding-standards.md).

---

## Step 11 — Build and package for distribution

`goco template:build` compiles and fingerprints assets, then writes the **generated** `manifest.json` with content hashes and resolved asset URLs. `goco template:package` produces the distributable, signed bundle.

```bash
# 1. Build: minify + fingerprint assets, emit manifest.json (never hand-edit the result)
goco template:build horizon

# 2. Package: create a signed .gocotpl artifact (Ed25519 over the manifest integrity hashes)
goco template:package horizon --sign --key ~/.goco/keys/marketplace-2026.key
#   -> dist/horizon-1.0.0.gocotpl
```

The generated `manifest.json` (do not edit):

```json
{
  "slug": "horizon",
  "version": "1.0.0",
  "builtAt": "2026-07-18T10:14:07Z",
  "engine": { "goco": ">=0.9 <1.0" },
  "adapter": "bootstrap@5.3",
  "integrity": {
    "template.json": "sha256-4d21…",
    "assets/css/app.css": "sha256-c07f…",
    "assets/js/app.js":  "sha256-9b2e…"
  },
  "assets": {
    "css/app.css": "/assets/t/horizon/app.4d21c0.css",
    "js/app.js":   "/assets/t/horizon/app.9b2ef1.js"
  },
  "locales": ["en", "es", "ta"],
  "signature": { "alg": "ed25519", "keyId": "marketplace-2026", "sig": "MEUCIQ…" }
}
```

Verify the artifact re-verifies against its signature before you publish:

```bash
goco template:verify dist/horizon-1.0.0.gocotpl
```

Publish to the marketplace (SemVer + Conventional Commits govern releases):

```bash
goco marketplace:publish dist/horizon-1.0.0.gocotpl --channel stable
```

Consumers install and pin it like any package:

```bash
composer require gococms/template-horizon:^1.0
goco template:install horizon
```

> **Warning**
> The installer refuses any template whose content hash does not match the signed `manifest.json`. Rebuild and re-sign after *any* change to source files — a hand-edited manifest or a stale hash fails verification on install. See the [Plugin Marketplace](../marketplace/overview.md) and [Security Model](../security/security-model.md).

---

## Step 12 — Ship a README

`README.md` is what marketplace users read first. Include: a one-line summary, a screenshot, the supported framework/version, the layouts and regions table, install commands, translation coverage, and a changelog stub that follows Conventional Commits.

```markdown
# Horizon

Marketing + blog template for Bootstrap 5.3 on GOCO CMS.

- Framework: Bootstrap 5.3 (`bootstrap` adapter)
- Layouts: default, landing, blog
- Locales: en, es, ta (RTL-ready)

## Install
```bash
composer require gococms/template-horizon:^1.0
goco template:install horizon
```

## Layouts & Regions
| Layout  | Regions |
| ------- | ------- |
| default | navbar, main, footer |
| landing | navbar, hero, features, cta, footer |
| blog    | navbar, posts, sidebar, footer |

## Changelog
### 1.0.0
- feat: initial Bootstrap 5 template with hero, cards, and paginated blog
```

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `template:install` rejects the bundle | Manifest hash mismatch after an edit | Re-run `template:build` then `template:package --sign`; never edit `manifest.json`. |
| Region renders empty | Region name in the layout view differs from `template.json` `regions` | Align the string in `$regions['...']` with the descriptor. |
| Grid columns don't stack on mobile | Bypassed the adapter with literal `col-*` classes | Use `$adapter->column($span, 12, $breakpoints)` so responsive variants emit. |
| String shows the raw key (e.g. `card.read_more`) | Missing translation key | Add the key to every catalog in `i18n.locales`; `template:validate` will flag gaps. |
| Edits not visible in preview | Template not linked | `goco template:link horizon`, then reload (no restart needed). |
| Fragment update reloads the whole page | Region not wrapped in `App::fragment()` | Wrap the htmx region so only its HTML is returned. |

---

## Related

- [Template Engine](../core/template-engine.md) — the engine that renders template packages
- [Theme Engine](../core/theme-engine.md) — how themes consume templates
- [Widget Guide](widget-guide.md) — building the widgets your renderers target
- [Theme Guide](theme-guide.md) — packaging a theme that references this template
- [Widget SDK](../sdk/widget-sdk.md) · [Theme SDK](../sdk/theme-sdk.md) · [Hook SDK](../sdk/hook-sdk.md) · [CLI SDK](../sdk/cli.md)
- [Rendering Pipeline](../architecture/rendering-pipeline.md) · [Request Lifecycle](../architecture/request-lifecycle.md) · [Event & Hook System](../architecture/event-hook-system.md)
- [Plugin Marketplace](../marketplace/overview.md) · [Security Model](../security/security-model.md) · [Project Structure](../getting-started/project-structure.md)
- [Documentation Index](../README.md)
