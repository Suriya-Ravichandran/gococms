# Comparison

> How GOCO CMS compares to WordPress, Webflow, Framer, Blogger, Wix Studio, Shopify, and the modern headless CMSs — an honest, architecture-first look at where GOCO fits and where it does not.

This page is written for people who already have a tool that works and are asking a fair question: *why would I move, and what would I be giving up?* GOCO is pre-1.0 and opinionated. It is not the right answer for everyone. Below we lay out the real tradeoffs, feature by feature, rather than a marketing scorecard where every row conveniently favors us.

> **Note**
> GOCO CMS is a **Website Operating System**: a small runtime core surrounded by an ecosystem of widgets, themes, and plugins. Its design goals are self-hostability, a coroutine-based PHP runtime ([ZealPHP](../architecture/zealphp-foundation.md) on OpenSwoole), a document database ([MongoDB](../architecture/database-mongodb.md)), first-class multi-tenancy, and a genuine visual [page builder](../core/page-builder.md). Keep those goals in mind — every comparison below is relative to *that* target, not to "best CMS in general."

---

## How To Read This Page

Three categories of product get compared here, and they answer different questions:

| Category | Products | The question they answer |
| --- | --- | --- |
| **Self-hosted open-source CMS** | WordPress, GOCO | "I want to own the whole stack and extend it in code." |
| **Hosted visual website builders** | Webflow, Framer, Wix Studio, Blogger | "I want to design and ship without running servers." |
| **Hosted commerce platform** | Shopify | "I want to sell products with the least operational risk." |
| **Headless / API-first CMS** | Directus, Strapi, Payload, Sanity | "I want a content backend and I'll build the frontend myself." |

GOCO's honest position: it is a **self-hosted, open-source, visually-editable Website Operating System** that also exposes a [headless API](../reference/api-reference.md). It overlaps WordPress most directly, borrows the visual-editing ambition of Webflow/Wix, and can be run headless like Directus/Strapi/Payload. That breadth is a strength *and* a risk — a young project spread across several product categories has more surface area to mature.

---

## Runtime & Architecture

This is where GOCO differs most, and where its biggest adoption friction lives.

| Dimension | GOCO | WordPress | Headless (Strapi/Payload/Directus) | Webflow / Wix / Framer / Blogger / Shopify |
| --- | --- | --- | --- | --- |
| Language | PHP 8.4+ | PHP 7.4+ | Node.js (TS/JS) | Proprietary (hosted) |
| Runtime model | **Persistent, coroutine-based** (OpenSwoole workers stay resident) | Per-request PHP-FPM (boot on every request) | Persistent Node event loop | Vendor-managed |
| Framework | [ZealPHP](https://github.com/sibidharan/zealphp) | Custom / plugin-driven | Express/Koa/Fastify-style | N/A |
| Concurrency | `go()`, coroutine channels, `co::sleep()`; non-blocking I/O in a request | Blocking; scale by adding FPM workers | Async via event loop | N/A |
| Primary proxy | **Traefik** (Docker-first, auto-HTTPS, HTTP/3) | Nginx/Apache typically | Nginx / platform LB | Vendor edge |
| Realtime | WebSocket + SSE **native to the runtime** | Bolt-on (external service or plugin) | Add-on / plugin | Vendor-provided |
| Deployment unit | Docker Compose stack | LAMP host or managed WP | Docker / PaaS | None (SaaS) |

The headline is the runtime. WordPress reboots PHP and reconnects to its database on essentially every request; that model is battle-tested, forgiving, and works on the cheapest shared hosting on earth. GOCO instead keeps long-lived OpenSwoole workers in memory, which makes in-process caching, connection pooling, WebSocket, SSE, background timers (`App::tick`), and cross-worker shared memory (`ZealPHP\Store`) natural rather than bolted on.

> **Warning**
> That runtime is exactly the barrier to entry. GOCO requires **OpenSwoole 22.1+** and the **ext-zealphp** PHP extension. You cannot drop GOCO onto $3/month cPanel shared hosting the way you can WordPress. A persistent worker model also means a fatal error or leaked resource can affect subsequent requests served by the same worker — code discipline matters more than in a fresh-boot-per-request world. If your team is not comfortable running Docker and a coroutine runtime, WordPress or a hosted builder will be less painful.

An async PHP runtime is genuinely unusual. Most PHP developers have never written coroutine code, and the mental model (never block a worker, avoid non-coroutine-aware blocking libraries) is a real learning cost. We think the payoff — one modern codebase that does realtime and background work without a second stack — is worth it, but it is a cost you pay up front. See [ZealPHP Foundation](../architecture/zealphp-foundation.md) and [Request Lifecycle](../architecture/request-lifecycle.md) for the details.

---

## Data Layer

| Dimension | GOCO | WordPress | Strapi | Payload | Directus | Sanity |
| --- | --- | --- | --- | --- | --- | --- |
| Primary store | **MongoDB** (document) | MySQL / MariaDB | SQL (PG/MySQL/SQLite) | MongoDB **or** SQL | SQL over existing DB | Proprietary (Content Lake) |
| Schema model | JSON-Schema validators per collection | Fixed `wp_posts` + serialized meta | Code/DB-defined models | Code-defined config | Introspects existing SQL | Portable Text / GROQ |
| Data access | Lightweight document-mapper + Repository ([Goco\Database](../architecture/database-mongodb.md)) | `WP_Query` + `$wpdb` | ORM (query engine) | ORM-ish | SQL abstraction | Hosted API + GROQ |
| Multi-doc transactions | Yes (MongoDB transactions) | Yes (InnoDB) | Yes | Yes | Yes | N/A (hosted) |
| Full-text search | Mongo text/Atlas, or [swappable providers](../architecture/search.md) (Meilisearch/OpenSearch) | MySQL `LIKE` / plugins | DB / plugin | DB | DB | Built-in |
| Versioning / soft delete / audit | Built into the [data model](../architecture/data-model.md) (`version`, `deleted_at`, `audit_logs`) | Post revisions; rest via plugins | Draft/publish | Versions | Revisions | Document history |

GOCO's choice of MongoDB is deliberate: a Website Operating System stores deeply nested, heterogeneous content (a page is a tree of Section → Container → Row → Column → Widget), and a document model maps to that tree without a forest of join tables. Every document carries `_id`, `created_at`, `updated_at`, `deleted_at`, `version`, `created_by`, `updated_by`, and tenant docs add `workspace_id` + `website_id`. See the full [Data Model](../architecture/data-model.md).

The honest tradeoff: MongoDB is less ubiquitous in the low-cost hosting market than MySQL, and a document store gives you weaker cross-entity referential guarantees than a relational schema — GOCO recovers invariants with explicit transactions and JSON-Schema validation rather than foreign keys. If your organization has deep SQL/relational operational expertise and no Mongo experience, that is a real cost to weigh. GOCO is a **document-mapper + Repository**, deliberately *not* a heavy ORM, so there is less magic but also fewer guardrails than a mature ORM provides.

---

## Visual Building & Editing

| Product | Visual builder | Model | Code escape hatch | Who owns the output |
| --- | --- | --- | --- | --- |
| **GOCO** | Yes — [Page Builder](../core/page-builder.md) over the Layout→Section→Container→Row→Column→Widget tree | Widgets are code (SDK), arranged visually | Full: PHP widgets, [templates](../core/template-engine.md), plugins | You (self-hosted, MIT) |
| WordPress | Gutenberg blocks; page builders via plugins (Elementor etc.) | Blocks / shortcodes | Full: PHP + JS | You (self-hosted, GPL) |
| Webflow | Best-in-class visual design | Designer maps to clean HTML/CSS | Limited (custom code embeds) | Vendor-hosted |
| Framer | Excellent, design-tool-native | React components under the hood | Code components | Vendor-hosted |
| Wix Studio | Strong, agency-oriented | Proprietary | Velo (JS) | Vendor-hosted |
| Blogger | Minimal template editor | Fixed blog templates | Theme XML/HTML | Vendor-hosted |
| Shopify | Theme editor (sections) | Liquid sections | Liquid + apps | Vendor-hosted |
| Strapi / Payload / Directus | **No page builder** (admin for structured data) | Content types | Code | You (self-hosted) |
| Sanity | Structured editing + Studio; page building via community plugins | Portable Text | Code | Vendor-hosted content lake |

Be fair here: **Webflow and Framer produce better visual design output today than GOCO's page builder does.** They are mature, focused tools with years of polish, and if pixel-perfect marketing-site design by non-developers is your single most important requirement, they are excellent — provided you accept SaaS hosting and lock-in. GOCO's differentiator is not "prettier than Webflow"; it is that the visual builder sits on top of a self-hosted, extensible, code-owned stack where every widget is a real [SDK](../sdk/widget-sdk.md) component you can version-control.

Against the headless CMSs the comparison flips entirely: Strapi, Payload, and Directus have **no visual page builder at all** — they are structured-content admins, and you build the presentation layer yourself in a separate frontend. GOCO gives you the page builder *and* the headless API in one system.

---

## Extensibility Model

| Product | Extension mechanism | Marketplace | Hook system |
| --- | --- | --- | --- |
| **GOCO** | [Widgets](../sdk/widget-sdk.md), [Themes](../sdk/theme-sdk.md), [Plugins](../sdk/plugin-sdk.md) via `Goco\SDK\*` facades | [Plugin Marketplace](../marketplace/overview.md) (early) | [Actions + Filters](../sdk/hook-sdk.md): `Hook::listen/dispatch/filter/apply` |
| WordPress | Plugins + themes (huge ecosystem) | Enormous, mature | Actions + filters (`add_action`/`add_filter`) — GOCO's model is deliberately familiar |
| Webflow | Apps + limited custom code | Growing | Vendor events |
| Framer | Code components, plugins | Growing | Vendor |
| Wix Studio | Velo APIs, app market | Mature | Vendor |
| Shopify | Apps, Liquid, Functions | Very large | Webhooks + Functions |
| Strapi | Plugins, lifecycle hooks | Moderate | Lifecycle hooks |
| Payload | Config + hooks (code-first) | Small | Field/collection hooks |
| Directus | Extensions, flows, hooks | Moderate | Event hooks + Flows |
| Sanity | Plugins, Studio customization | Moderate | Webhooks |

GOCO's hook system is intentionally close to WordPress's mental model — actions (`subject.verb`, e.g. `page.rendered`, `content.published`) and filters (`subject.noun`, e.g. `page.title`, `menu.items`) — so the millions of developers who understand `add_action`/`add_filter` can transfer that intuition directly. The difference is that GOCO's hooks are namespaced, typed, and can dispatch asynchronously on coroutines (`Hook::dispatchAsync`). See the [Hook SDK](../sdk/hook-sdk.md) and [Event & Hook System](../architecture/event-hook-system.md).

The unavoidable truth: **WordPress's ecosystem is the largest in the CMS world by a wide margin.** Tens of thousands of plugins, themes, and integrations, plus a global talent pool. GOCO's marketplace is new. If your requirement can be satisfied by an existing WordPress plugin today, that is a legitimate reason to stay on WordPress. GOCO bets on a cleaner extension SDK and a modern runtime over ecosystem size — a bet that only pays off over time.

---

## Multi-Tenancy

| Product | Multi-tenancy | How |
| --- | --- | --- |
| **GOCO** | **First-class** | Workspace → Website hierarchy; `workspace_id` + `website_id` on tenant docs (default), optional database-per-workspace (enterprise); per-tenant [Traefik](../deployment/traefik.md) routers. See [Multi-Tenancy](../architecture/multi-tenancy.md). |
| WordPress | Multisite (bolted on) | Shared tables with `blog_id` prefixing; historically operationally awkward |
| Webflow / Wix / Framer | Per-site accounts | Not true tenant isolation for *your* customers |
| Shopify | One store per account | Plus (org-level) for multi-store |
| Strapi / Payload / Directus | Not built-in | You build it yourself |
| Sanity | Datasets / projects | Project-scoped |

Multi-tenancy is a core GOCO design axis, not an afterthought. If you are an agency or SaaS operator running many websites for many customers under one system, this is one of GOCO's strongest arguments versus both WordPress Multisite (which many operators find painful at scale) and the headless CMSs (which leave tenancy entirely to you). See [Multi-Tenancy](../architecture/multi-tenancy.md) and the [Permission System](../architecture/permission-system.md) for how RBAC + optional ABAC scope per `(workspace, website)`.

---

## Hosting, Licensing & Cost

| Product | Hosting | License | Vendor lock-in | Data ownership |
| --- | --- | --- | --- | --- |
| **GOCO** | Self-hosted (Docker) | **MIT** | Low (own the stack) | Full |
| WordPress | Self-host or managed | GPLv2+ | Low | Full |
| Webflow | Hosted only | Proprietary | High | Export limited |
| Framer | Hosted only | Proprietary | High | Export limited |
| Wix Studio | Hosted only | Proprietary | High | Limited export |
| Blogger | Hosted (free) | Proprietary | High | Export via Blogger API |
| Shopify | Hosted only | Proprietary | High | Product/order export |
| Strapi | Self-host or Cloud | Enterprise-tiered OSS | Low–medium | Full |
| Payload | Self-host or Cloud | MIT | Low | Full |
| Directus | Self-host or Cloud | BSL/commercial tiers | Low–medium | Full |
| Sanity | Hosted content lake | Proprietary backend + OSS Studio | Medium–high | Via API |

GOCO is **MIT-licensed and self-hosted** — you own the code, the data, and the deployment. There is no per-seat SaaS bill and no vendor that can change pricing under you. That freedom comes with the operational responsibility a hosted builder removes: you run MongoDB, Redis, Traefik, and the GOCO stack yourself (see [Docker Architecture](../deployment/docker.md) and the [Deployment Guide](../deployment/deployment-guide.md)). "Free as in freedom" is not "free as in no ops."

---

## Learning Curve & Team Fit

| Product | Best for | Learning curve |
| --- | --- | --- |
| Blogger | Individuals wanting a free blog now | Trivial |
| Wix Studio / Framer / Webflow | Designers & agencies, no ops | Low–moderate (design tools) |
| Shopify | Merchants who must not lose sales to downtime | Low |
| WordPress | Everyone; largest talent pool | Low to start, deep to master |
| Strapi / Directus | Node teams wanting a content backend | Moderate |
| Payload / Sanity | Code-first teams building bespoke frontends | Moderate–high |
| **GOCO** | Teams wanting a self-hosted OS for *many* sites, comfortable with Docker + a modern PHP async runtime | **Moderate–high (runtime is novel)** |

GOCO asks more of you than a hosted builder and, right now, more than WordPress — because the coroutine runtime is unfamiliar and the ecosystem is young. It rewards teams that value owning a modern, unified stack over convenience.

---

## Where GOCO Fits — and Where It Doesn't

**Choose GOCO when:**

- You want to **self-host** and fully **own** your stack (MIT, no SaaS lock-in).
- You run **many websites for many tenants** (agency / SaaS) and want real multi-tenancy, not Multisite workarounds.
- You want a **visual page builder *and* a headless API** in one system.
- You want **native realtime** (WebSocket/SSE), background timers, and in-process caching without a second runtime.
- Your team is comfortable with **Docker, MongoDB, and modern PHP**, and is willing to learn a coroutine model.

**Do NOT choose GOCO (yet) when:**

- You need **rock-solid, today** stability for revenue-critical commerce — **Shopify** removes operational risk that a pre-1.0 self-hosted platform cannot match.
- **Pixel-perfect visual design by non-developers** is your single top requirement — **Webflow** or **Framer** are more polished design tools today.
- You want the **absolute cheapest, click-to-install** hosting with a plugin for everything — **WordPress** on shared hosting still wins on ubiquity and ecosystem size.
- You cannot run **OpenSwoole / ext-zealphp / Docker** and need pure managed SaaS with zero ops — a hosted builder or Sanity fits better.
- You only need a **content API** for a frontend you're already building in Node/React and want no page builder — **Payload** or **Directus** may be a leaner fit.

> **Tip**
> GOCO can also be adopted **headless** — expose the [REST/JWT API](../reference/api-reference.md) and drive your own frontend — while still keeping the option to turn on the visual builder later. You do not have to pick "visual" vs "headless" up front the way you do with most tools in this table.

### Pre-1.0 Reality Check

GOCO is pre-1.0 and under active development. It follows [Semantic Versioning](../roadmap.md), which means the public API surface (SDK facades, hook names, data model) is stabilizing but **may still change before 1.0**. Some subsystems are `stable`, others `beta` or `experimental` — check the stability tag at the top of each doc and the [Roadmap](../roadmap.md) before betting a production launch on a given feature. If you need a frozen, contract-guaranteed platform for a mission-critical launch next quarter, that is a legitimate reason to wait for 1.0 or choose a mature alternative for now.

---

## Migrating From WordPress

WordPress is GOCO's closest analog and the most common source system, so a migration path matters. This is a **conceptual** overview; see the migration tooling in the [CLI Reference](../reference/cli-reference.md) for the concrete commands.

### What Maps Cleanly

| WordPress concept | GOCO equivalent | Notes |
| --- | --- | --- |
| Post / Page | `posts` / `pages` collections | Body content is re-parsed into the widget tree where possible |
| Post revisions | `post_revisions` / `page_revisions` | Native versioning |
| Categories / Tags | `taxonomies` + `terms` + `term_relationships` | Direct conceptual mapping |
| Media library | `media` collection + [Storage](../architecture/storage.md) driver | Files re-homed to Local/MinIO/S3 |
| Users / Roles | `users` + `roles` | WP roles map onto GOCO's [hierarchical roles](../architecture/permission-system.md) |
| Menus | `menus` collection | Structure preserved |
| Redirects / permalinks | `redirects` collection | Import to preserve SEO ([SEO](../core/blog-engine.md)) |
| `add_action` / `add_filter` | `Hook::listen` / `Hook::filter` | Familiar mental model, new signatures |

### What Requires Real Work

- **PHP plugins do not port automatically.** WordPress plugins assume the WP runtime, `$wpdb`/MySQL, and the FPM request model. A GOCO plugin uses the [Plugin SDK](../sdk/plugin-sdk.md), MongoDB repositories, and coroutine-safe code. Custom plugin logic must be **rewritten**, not imported.
- **Themes are re-implemented.** WordPress themes (PHP template hierarchy + `functions.php`) become GOCO [themes](../sdk/theme-sdk.md) (manifest + layouts/regions) and [widgets](../sdk/widget-sdk.md). Layout intent transfers; template code does not.
- **Serialized meta & shortcodes** need mapping. `wp_postmeta` serialized blobs and shortcodes must be translated into structured fields or widgets.
- **Page-builder content** (Elementor/Divi/etc.) is stored in proprietary formats; expect a lossy or manual conversion into GOCO's widget tree.
- **Database engine changes** from MySQL to MongoDB — an ETL step, not a dump/restore.

### Suggested Migration Sequence

```bash
# 1. Stand up a GOCO stack (see the Deployment Guide)
docker compose up -d

# 2. Export the WordPress site (WXR + DB) using standard WP tooling,
#    then run the GOCO importer against it
goco migrate:wordpress \
  --source ./wordpress-export.xml \
  --db-dsn "mysql://user:pass@old-host/wp" \
  --media-root ./wp-content/uploads \
  --workspace acme --website acme-main \
  --dry-run

# 3. Review the dry-run report, then apply
goco migrate:wordpress --source ./wordpress-export.xml ... --apply

# 4. Re-point DNS through Traefik once verified
```

```mermaid
flowchart LR
  A[WordPress<br/>MySQL + wp-content] -->|export WXR + DB dump| B[goco migrate:wordpress]
  B --> C{Dry-run report}
  C -->|content maps| D[MongoDB collections<br/>pages / posts / media / terms]
  C -->|plugins & themes| E[Rewrite in GOCO SDK]
  D --> F[Verify on staging domain]
  E --> F
  F -->|cutover DNS via Traefik| G[Live on GOCO]
```

> **Warning**
> Always run `--dry-run` first and migrate into a **staging** website (a separate `website_id`) before cutover. Budget time for rewriting custom plugins and themes — for a heavily customized WordPress site, that rewrite, not the content import, is the bulk of the effort. Do not expect a one-click, zero-loss migration; no honest tool can promise that across two different runtimes and two different databases.

---

## Summary Verdict

GOCO is not trying to beat every tool on this page at that tool's own game. It will not out-design Webflow, out-ecosystem WordPress, or out-de-risk Shopify **today**. What it offers is a specific, coherent bet: a **self-hosted, MIT-licensed Website Operating System** with a modern coroutine PHP runtime, a document database, genuine multi-tenancy, and a visual builder that sits on a fully code-owned, extensible stack. If that combination is what you want and you can run the runtime, GOCO is compelling. If it isn't, this page should have told you honestly which of the alternatives to reach for instead.

---

## Related

- [Overview](./overview.md)
- [Philosophy & Design Principles](./philosophy.md)
- [Architecture Overview](../architecture/overview.md)
- [ZealPHP Foundation](../architecture/zealphp-foundation.md)
- [MongoDB Data Layer](../architecture/database-mongodb.md)
- [Multi-Tenancy](../architecture/multi-tenancy.md)
- [Page Builder (Visual Editor)](../core/page-builder.md)
- [Plugin SDK](../sdk/plugin-sdk.md)
- [CLI Reference](../reference/cli-reference.md)
- [Roadmap](../roadmap.md)
- [Documentation Index](../README.md)
