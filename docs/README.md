# GOCO CMS Documentation

> **GOCO CMS** — The Open Source Website Operating System.
> A modern, enterprise-grade website builder and CMS powered by [ZealPHP](https://github.com/sibidharan/zealphp) + OpenSwoole, backed by MongoDB and Redis, and shipped Docker-first behind Traefik.

Welcome to the official documentation for GOCO CMS. This is the canonical reference for
everyone who **uses**, **extends**, or **contributes to** the platform. New here? Start with
**[Introduction → Overview](introduction/overview.md)**, then the **[Quick Start](getting-started/quick-start.md)**.

---

## What is GOCO CMS?

GOCO CMS combines the simplicity of Google Blogger, the flexibility of WordPress, and the
visual editing experience of Webflow/Framer — running on a **persistent, async PHP runtime**
(ZealPHP on OpenSwoole) instead of the classic request-per-process model.

It is not "a CMS." It is a **Website Operating System**: a lightweight core surrounded by a
first-class ecosystem of **widgets**, **themes**, and **plugins**, every one of which is
replaceable, versioned, and community-extendable. Build blogs, business sites, landing pages,
docs sites, portals, LMS, and eCommerce — visually, without writing HTML/CSS/JS.

### The stack at a glance

| Layer | Technology |
| --- | --- |
| Runtime | **ZealPHP** on **OpenSwoole 22.1+**, **PHP 8.4+** (persistent workers, coroutines, WebSocket, SSE) |
| Primary database | **MongoDB** (collections, JSON-Schema validation, aggregation, transactions, audit logs) |
| Cache / queue / realtime | **Redis** (cache, sessions, queue, pub/sub, rate limiting, locks) |
| Object storage | Driver interface — **Local**, **MinIO**, **Amazon S3** |
| Search | Provider interface — **MongoDB**, **Meilisearch**, **OpenSearch** |
| Edge / proxy | **Traefik** (automatic HTTPS, wildcard/multi-domain, HTTP/3) |
| Frontend / UI | **htmx** hypermedia over server-rendered HTML — `App::fragment()` regions + SSE, progressively enhanced; no SPA framework required for content or admin surfaces |
| Packaging | **Docker-first** modular monorepo |

---

## Documentation map

### 1. Introduction
- [Overview](introduction/overview.md) — the vision, the product, and who it is for
- [Philosophy & Design Principles](introduction/philosophy.md) — the rules that govern every decision
- [Comparison](introduction/comparison.md) — GOCO vs WordPress, Webflow, Blogger, and headless CMSs

### 2. Product
- [Product Requirements Document (PRD)](product/prd.md) — problem, personas, epics, success metrics
- [Software Requirements Specification (SRS)](product/srs.md) — numbered functional & non-functional requirements
- [Domain Model (DDD)](product/domain-model.md) — bounded contexts, aggregates, domain events

### 3. Getting Started
- [Installation](getting-started/installation.md) — requirements and install paths (Docker, Composer, source)
- [Quick Start](getting-started/quick-start.md) — from zero to a published page in ~10 minutes
- [Project Structure](getting-started/project-structure.md) — the modular monorepo, folder by folder
- [Configuration](getting-started/configuration.md) — environment, config files, per-workspace settings

### 4. Architecture
- [Architecture Overview](architecture/overview.md) — the master system diagram and module map
- [ZealPHP Foundation](architecture/zealphp-foundation.md) — how GOCO uses the async runtime
- [Request Lifecycle](architecture/request-lifecycle.md) — what happens on every request
- [Service Container & DI](architecture/service-container.md) — how services are wired
- [Event & Hook System](architecture/event-hook-system.md) — actions, filters, and the event bus
- [MongoDB Data Layer](architecture/database-mongodb.md) — ODM, repositories, validation, transactions
- [Data Model](architecture/data-model.md) — the canonical collection & index catalog
- [Permission System](architecture/permission-system.md) — roles, capabilities, and policies (RBAC + ABAC)
- [Rendering Pipeline](architecture/rendering-pipeline.md) — how a page becomes HTML
- [Multi-Tenancy](architecture/multi-tenancy.md) — workspaces, websites, custom domains
- [Caching, Queue & Realtime](architecture/caching-and-queue.md) — Redis, jobs, pub/sub
- [Storage & Media](architecture/storage.md) — pluggable object storage drivers
- [Search](architecture/search.md) — pluggable search providers

### 5. Core Modules
Deep dives into the modules that ship in the lightweight core. Each follows the
[module documentation standard](#module-documentation-standard).

- [Authentication](core/authentication.md)
- [Routing](core/routing.md)
- [Widget Engine](core/widget-engine.md)
- [Theme Engine](core/theme-engine.md)
- [Template Engine](core/template-engine.md)
- [Plugin Engine](core/plugin-engine.md)
- [Page Builder (Visual Editor)](core/page-builder.md)
- [Blog Engine](core/blog-engine.md)
- [Database Builder (Dynamic Collections)](core/database-builder.md)
- [AI Platform](core/ai-platform.md) — `experimental`

### 6. SDKs
The stable, public APIs you build against. These follow [semantic versioning](community/governance.md#versioning).

- [Widget SDK](sdk/widget-sdk.md)
- [Theme SDK](sdk/theme-sdk.md)
- [Plugin SDK](sdk/plugin-sdk.md)
- [Hook SDK](sdk/hook-sdk.md)
- [CLI SDK](sdk/cli.md)

### 7. Developer Guides
Practical, end-to-end tutorials.

- [Widget Guide](guides/widget-guide.md) — build a widget package
- [Theme Guide](guides/theme-guide.md) — build a Tailwind theme
- [Plugin Guide](guides/plugin-guide.md) — build and ship a plugin
- [Template Guide](guides/template-guide.md) — package a template (Bootstrap, Tailwind, Bulma…)

### 8. Reference
- [API Reference](reference/api-reference.md) — every public facade, class, and HTTP endpoint
- [Configuration Reference](reference/configuration-reference.md) — every config key and env var
- [CLI Reference](reference/cli-reference.md) — every `goco` command

### 9. Deployment & Operations
- [Docker Architecture](deployment/docker.md) — images, compose files, services
- [Traefik Reverse Proxy](deployment/traefik.md) — TLS, routing, HTTP/3, multi-tenant
- [Deployment Guide](deployment/deployment-guide.md) — production rollout and upgrades
- [Backup & Restore](deployment/backup-restore.md) — DR strategy per data store
- [Scaling](deployment/scaling.md) — single-node to cluster

### 10. Ecosystem & Security
- [Plugin Marketplace](marketplace/overview.md) — publishing and installing extensions
- [Security Model](security/security-model.md) — threat model, hardening, disclosure

### 11. Community & Project
- [Contributing](community/contributing.md)
- [Code of Conduct](community/code-of-conduct.md)
- [Governance](community/governance.md)
- [Support](community/support.md)
- [Coding Standards](community/coding-standards.md)
- [Testing Strategy](community/testing-strategy.md)
- [Roadmap](roadmap.md)
- [Changelog](changelog.md)
- [Glossary](glossary.md)

---

## Reading paths

Not sure where to start? Pick the path that matches your goal.

| I want to… | Read, in order |
| --- | --- |
| **Build a website** without code | [Overview](introduction/overview.md) → [Quick Start](getting-started/quick-start.md) → [Page Builder](core/page-builder.md) → [Theme Guide](guides/theme-guide.md) |
| **Build a widget** | [Widget Engine](core/widget-engine.md) → [Widget SDK](sdk/widget-sdk.md) → [Widget Guide](guides/widget-guide.md) |
| **Build a plugin** | [Plugin Engine](core/plugin-engine.md) → [Plugin SDK](sdk/plugin-sdk.md) → [Hook SDK](sdk/hook-sdk.md) → [Plugin Guide](guides/plugin-guide.md) |
| **Build a theme/template** | [Theme Engine](core/theme-engine.md) → [Theme SDK](sdk/theme-sdk.md) → [Template Guide](guides/template-guide.md) |
| **Model custom data** | [Database Builder](core/database-builder.md) → [MongoDB Data Layer](architecture/database-mongodb.md) → [Data Model](architecture/data-model.md) |
| **Understand the internals** | [Architecture Overview](architecture/overview.md) → [ZealPHP Foundation](architecture/zealphp-foundation.md) → [Request Lifecycle](architecture/request-lifecycle.md) |
| **Deploy to production** | [Installation](getting-started/installation.md) → [Docker](deployment/docker.md) → [Traefik](deployment/traefik.md) → [Deployment Guide](deployment/deployment-guide.md) |
| **Contribute to core** | [Architecture Overview](architecture/overview.md) → [Contributing](community/contributing.md) → [Coding Standards](community/coding-standards.md) → [Governance](community/governance.md) |

---

## Module documentation standard

Every core module and every marketplace-grade extension is documented against the same
17-section template so readers always know where to find things:

1. **Purpose** — what problem it solves.
2. **Functional Specification** — what it does, precisely.
3. **Business Requirements** — the outcomes it must guarantee.
4. **User Stories** — `As a <role>, I want <capability>, so that <benefit>`.
5. **Data Model** — MongoDB collections, fields, and indexes.
6. **Folder Structure** — where the code lives (which monorepo package).
7. **API Design** — public classes, facades, and signatures.
8. **Services** — the container bindings it registers.
9. **Events** — actions it dispatches.
10. **Hooks** — filters it exposes.
11. **UI Architecture** — admin and front-end surfaces.
12. **Security Model** — trust boundaries, permissions, validation.
13. **Performance Strategy** — caching, async, and hot paths.
14. **Testing Strategy** — unit, integration, and end-to-end coverage.
15. **Extension Points** — how others build on it.
16. **Upgrade Strategy** — migrations and backward compatibility.
17. **Future Roadmap** — planned evolution.

---

## Conventions used throughout these docs

- **Stability tags** — `stable`, `beta`, `experimental`, and `deprecated` mark every public API.
  Only `stable` APIs are covered by our [backward-compatibility promise](community/governance.md#backward-compatibility).
- **Code language** — runtime code is **PHP 8.4+** on **ZealPHP**; shell commands use `bash`. The developer CLI is `goco`; the runtime process CLI is `php app.php`.
- **Namespaces** — the async runtime is `ZealPHP\…`; GOCO core is `Goco\…`; public facades live in `Goco\SDK\…`.
- **Data** — the primary database is **MongoDB**; collection names are `snake_case` plural (see [Data Model](architecture/data-model.md)).
- **Callouts** — `> **Note**`, `> **Warning**`, and `> **Tip**` blocks flag things worth pausing on.

---

## Status of this documentation

GOCO CMS is pre-1.0 and under active development; APIs marked `experimental` may change between
minor versions. Documentation is maintained alongside the code — if you find a gap or an error,
see [Contributing → Documentation](community/contributing.md#contributing-documentation).

**License:** GOCO CMS and its documentation are released under the **MIT License**.
