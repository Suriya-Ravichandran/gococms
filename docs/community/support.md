# Support

> How to get help with GOCO CMS — the right channel for questions, bug reports, security disclosures, and commercial support, plus how to ask so you get a fast, useful answer.

GOCO CMS is an MIT-licensed, community-driven **Website Operating System** built on [ZealPHP](https://github.com/sibidharan/zealphp), OpenSwoole, PHP 8.4+, MongoDB, and Redis. Because the project is pre-1.0 and under active development, help is offered through several channels — each optimized for a different kind of question. This page tells you which one to use, what to include, and what you can reasonably expect back.

> **Note**
> **Community support carries no SLA.** It is provided by volunteers on a best-effort basis. If you need guaranteed response times, see [Commercial & Enterprise Support](#commercial--enterprise-support) below.

---

## Where to Get Help (in order)

Work down this list. Most questions are answered fastest at step 1 or 2, and reserving GitHub Issues for genuine bugs keeps the tracker signal high for everyone.

| # | Channel | Best for | Response expectation |
|---|---------|----------|----------------------|
| 1 | **Documentation** ([docs root](../README.md)) | Setup, configuration, API, SDK, how-to | Instant — self-serve |
| 2 | **GitHub Discussions** (Q&A) | "How do I…?", design questions, showcase | Hours to days (community) |
| 3 | **Community Chat** (Discord / Matrix) | Real-time help, quick pointers, pairing | Real-time, no guarantee |
| 4 | **Stack Overflow** (`gococms` tag) | Reusable, searchable Q&A | Community-driven |
| 5 | **GitHub Issues** | Confirmed **bugs** and **feature requests only** | Triage in days |
| — | **Security disclosures** | Vulnerabilities — **private only** | See [Security](#security-issues) |
| — | **Commercial support** | SLAs, migrations, custom work | Contractual |

### 1. Documentation (start here)

The [documentation](../README.md) is the fastest source of truth and is versioned alongside the code. Before asking anywhere else, search it:

- **Installation & setup:** [Installation](../getting-started/installation.md), [Quick Start](../getting-started/quick-start.md), [Configuration](../getting-started/configuration.md)
- **How things work:** [Architecture Overview](../architecture/overview.md), [ZealPHP Foundation](../architecture/zealphp-foundation.md), [Request Lifecycle](../architecture/request-lifecycle.md)
- **Building on GOCO:** [Widget SDK](../sdk/widget-sdk.md), [Theme SDK](../sdk/theme-sdk.md), [Plugin SDK](../sdk/plugin-sdk.md), [Hook SDK](../sdk/hook-sdk.md), [CLI SDK](../sdk/cli.md)
- **Reference:** [API Reference](../reference/api-reference.md), [Configuration Reference](../reference/configuration-reference.md), [CLI Reference](../reference/cli-reference.md)
- **Deploying:** [Docker Architecture](../deployment/docker.md), [Traefik Reverse Proxy](../deployment/traefik.md), [Deployment Guide](../deployment/deployment-guide.md)
- **Terminology:** [Glossary](../glossary.md)

> **Tip**
> Found a documentation gap, typo, or something that's out of date? That *is* a valid GitHub Issue (label `documentation`) or, better, a pull request. See [Contributing](./contributing.md).

### 2. GitHub Discussions (community Q&A)

For usage questions, design guidance, "is this the right approach?" conversations, and showing off what you built, use **GitHub Discussions** on the [`gococms/core`](https://github.com/gococms/core) repository.

Categories:

- **Q&A** — ask a question; the community (and maintainers) answer, and the best reply can be marked as the accepted answer.
- **Ideas** — float a feature or direction before it becomes a formal proposal.
- **Show and tell** — share sites, plugins, themes, and widgets built on GOCO.
- **General** — everything else.

Discussions are preferred over Issues for anything that isn't a confirmed defect or a concrete feature request, because they keep the issue tracker focused and are easier for the next person to find.

### 3. Community Chat (Discord / Matrix)

Real-time help, quick questions, and pairing happen in community chat.

- **Discord:** the community chat invite is published in the [repository README](https://github.com/gococms/core).
- **Matrix:** `#gococms:matrix.org`, bridged to the Discord community.

Chat is great for unblocking quickly, but it is ephemeral. If your question and its answer would help others, please also post it as a **Discussion** or on **Stack Overflow** so it's searchable. Never paste secrets (API keys, JWT signing secrets, `.env` contents, MongoDB/Redis credentials) into chat.

### 4. Stack Overflow (`gococms` tag)

For self-contained, reusable programming questions, ask on **Stack Overflow** and tag it [`gococms`](https://stackoverflow.com/questions/tagged/gococms). Add secondary tags where relevant: `php`, `openswoole`, `mongodb`, `redis`, `zealphp`. Stack Overflow's format is ideal for questions like "how do I register a widget with a property schema?" that have a concrete, code-shaped answer.

---

## Reporting Bugs & Requesting Features (GitHub Issues)

> **Warning**
> GitHub Issues are **only** for confirmed bugs and concrete feature requests. Usage questions belong in [Discussions](#2-github-discussions-community-qa). Security vulnerabilities must go through the [private channel](#security-issues) — never a public issue.

Open issues on the relevant repository (usually [`gococms/core`](https://github.com/gococms/core); use the specific package repo — `gococms/cli`, a `packages/*` repo, etc. — when you know the fault lives there). Search existing open and closed issues first; a duplicate that adds a reproduction is far more useful than a fresh thread.

### What to include in a bug report

A good bug report is reproducible. Include:

1. **A minimal reproduction.** The smallest steps, config, or repo that triggers the bug. Reproductions are the single biggest factor in how fast a bug gets fixed.
2. **Expected vs. actual behavior.** What you thought would happen, and what did.
3. **Environment / versions.** Gather these with:

   ```bash
   goco --version              # GOCO CLI + core version
   php -v                      # PHP 8.4+ expected
   php --ri openswoole         # OpenSwoole 22.1+ expected
   docker compose version
   git -C . rev-parse --short HEAD   # commit, since pre-1.0 moves fast
   ```

   Also note: OS/host, MongoDB and Redis versions, whether you run via Docker Compose or bare metal, and the ZealPHP process mode (`MODE_COROUTINE`, `MODE_LEGACY_CGI`, etc.).

4. **Logs.** ZealPHP writes logs to `/tmp/zealphp/`. Attach the relevant portion:

   ```bash
   php app.php logs            # tail the running process logs
   ls -la /tmp/zealphp/        # locate log files
   docker compose logs gococms --tail=200   # container logs
   ```

   Include MongoDB/Redis errors and Traefik router logs when the fault is in routing or TLS.
5. **Config (redacted).** The relevant `.env` keys and `docker-compose.yml` service block — with all secrets removed.
6. **Scope.** Does it reproduce on a clean install? Which workspace/website/tenant? Regression from a known-good commit?

> **Tip**
> Use `git bisect` between a working commit and a broken one to pinpoint the change. A bisected commit hash turns hours of triage into minutes.

### What to include in a feature request

- The **problem** you're trying to solve (not just the solution you have in mind).
- Who it helps and how often it comes up.
- Whether it belongs in **core**, a **package**, or a **plugin/widget/theme** — GOCO deliberately keeps the core lightweight and pushes features to the ecosystem. See the [Architecture Overview](../architecture/overview.md) and [Plugin Marketplace](../marketplace/overview.md).
- Any prior art, alternatives considered, and rough API sketch.

Larger changes may be asked to go through the proposal / RFC process described in [Governance](./governance.md).

---

## How to Ask a Good Question

The quality of the answer tracks the quality of the question. Whatever the channel:

- **Search first.** Docs, closed issues, Discussions, and Stack Overflow. Link what you already found and why it didn't fit.
- **State the goal, not just the blocker.** The [XY problem](https://xyproblem.info/) — asking about your attempted fix instead of the real objective — wastes everyone's time.
- **Make it reproducible and minimal.** Trim to the smallest snippet that still shows the problem. Ground runtime code in the real ZealPHP API, e.g.:

  ```php
  $app->route('/hello/{name}', function ($name, $request, $response) {
      return ['hello' => $name]; // array -> JSON automatically
  });
  ```

- **Show what you tried** and the exact error text (copy-paste, don't screenshot logs).
- **Post text, not images, for code and logs.** Use fenced code blocks with a language tag so it's searchable and copyable.
- **Give versions up front** (see the [commands above](#what-to-include-in-a-bug-report)).
- **One question per thread.** Bundled questions get partial answers.
- **Be patient and courteous.** Everyone here is a volunteer; follow the [Code of Conduct](./code-of-conduct.md).
- **Close the loop.** When something works, post the solution and mark the accepted answer. The next person searching will thank you.

---

## Community Expectations

- **No SLA.** Community help is best-effort by volunteers across time zones. There is no guaranteed response time on Discussions, chat, Stack Overflow, or the issue tracker.
- **Maintainer triage is prioritized**, not first-come-first-served. Reproducible bugs, security fixes, and regressions jump the queue; vague or unreproducible reports may be closed with a request for more detail.
- **Pre-1.0 means change.** APIs, config keys, and internals can shift between releases under [Semantic Versioning](../glossary.md). Always report the exact version/commit.
- **Be a good citizen.** Give back where you can — answer a Discussion, improve a doc, add a reproduction to an existing issue. GOCO is sustained by [contributors](./contributing.md), and mutual respect under the [Code of Conduct](./code-of-conduct.md) is non-negotiable.
- **Project direction** is decided in the open per the [Governance](./governance.md) model.

---

## Commercial & Enterprise Support

Community channels are not a substitute for a support contract. For teams that need guarantees, GOCO's commercial ecosystem offers:

- **SLA-backed support** — defined response and resolution targets, priority triage, private issue handling.
- **Enterprise deployment help** — multi-tenant scaling ([Scaling Strategy](../deployment/scaling.md)), high-availability MongoDB/Redis, Traefik multi-domain and HTTP/3 rollout ([Traefik](../deployment/traefik.md)), backup and DR ([Backup & Restore](../deployment/backup-restore.md)).
- **Migrations** — moving from other CMS platforms onto GOCO, including data and theme conversion.
- **Custom development** — bespoke widgets, themes, plugins, and integrations built against the [SDK](../sdk/plugin-sdk.md).
- **Training & architecture reviews.**

> **Note**
> Commercial support and the certified **partner directory** are being established. Contact and partner details will be published on the project website and the [repository README](https://github.com/gococms/core). Vendors interested in becoming partners should watch [Governance](./governance.md) and the [Marketplace overview](../marketplace/overview.md).

---

## Security Issues

> **Warning**
> **Never report a security vulnerability in a public GitHub issue, Discussion, chat, or Stack Overflow post.** Public disclosure before a fix ships puts every GOCO deployment at risk.

Report vulnerabilities privately through the coordinated-disclosure process in the [Security Model](../security/security-model.md) — typically GitHub's **private security advisories** on the affected repository, or the dedicated security contact listed there. You'll get an acknowledgement, a coordinated fix, and credit in the advisory once a patch is available.

This covers authentication and session handling (Redis sessions, JWT, OAuth2, 2FA/TOTP, WebAuthn passkeys, Argon2id, CSRF), RBAC/ABAC permission bypasses, multi-tenant isolation (`workspace_id` / `website_id` leakage), injection, SSRF, and any issue that could compromise a deployment. See [Security Model](../security/security-model.md) for scope and full details.

---

## Roadmap & Project Status

Before asking "is X planned?" or "when will Y land?", check the public status pointers:

- **[Roadmap](../roadmap.md)** — planned direction and priorities.
- **[Changelog](../changelog.md)** — what shipped in each release (Semantic Versioning, Conventional Commits).
- **GitHub Milestones & Projects** on [`gococms/core`](https://github.com/gococms/core) — in-flight work and release planning.
- **[Governance](./governance.md)** — how decisions and proposals are made, and how to influence direction.

If your question is "why isn't this done yet?", the roadmap and open milestones usually answer it — and if the work isn't tracked anywhere, that's a great prompt for a [feature request](#what-to-include-in-a-feature-request) or a Discussion.

---

## Related

- [Documentation home](../README.md)
- [Contributing](./contributing.md)
- [Code of Conduct](./code-of-conduct.md)
- [Governance](./governance.md)
- [Coding Standards](./coding-standards.md)
- [Testing Strategy](./testing-strategy.md)
- [Security Model](../security/security-model.md)
- [Roadmap](../roadmap.md)
- [Changelog](../changelog.md)
- [Plugin Marketplace](../marketplace/overview.md)
- [Glossary](../glossary.md)
