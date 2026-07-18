# AI Platform

> The AI Platform is GOCO CMS's provider-agnostic generative layer — website, page, theme, widget, blog, SEO, accessibility, code, translation, and image generation — built on a swappable `AIProvider` driver interface that defaults to Anthropic Claude, streams token-by-token over SSE/WebSocket, and enforces guardrails, quotas, and cost controls per tenant.

**Stability:** `experimental` — the entire `packages/ai` surface (facades, events, provider contract, `ai_jobs` schema) may change before 1.0. Pin `gococms/ai` and read the [Changelog](../changelog.md) before upgrading. Individual capabilities are tagged below.

The AI Platform is the twelfth core module of the "Website Operating System." It is a lightweight orchestration layer around the same primitives every other module uses: [MongoDB](../architecture/database-mongodb.md) for durable jobs and audit, [Redis](../architecture/caching-and-queue.md) for queues, rate limits, quotas, and realtime pub/sub, [OpenSwoole task workers](../architecture/zealphp-foundation.md) for asynchronous generation, and the [Hook system](../architecture/event-hook-system.md) so plugins can register their own AI actions. Nothing about the platform is bound to a single model vendor — Claude is the recommended default, not a hard dependency.

---

## 1. Purpose

Content and structure creation is the highest-friction task in any CMS. The AI Platform removes that friction by turning natural-language intent into first-class GOCO artifacts — a `websites` document, a `pages` tree of Sections/Rows/Columns/Widgets, a registered [Theme](theme-engine.md), a generated [Widget](widget-engine.md) definition, a published [blog post](blog-engine.md), a set of `redirects`, or an accessibility report.

Concretely, the module exists to:

- **Generate structure, not just prose.** Output conforms to the GOCO domain model (Workspace → Website → Theme → Layout → Section → Container → Row → Column → Widget) and is validated against the same JSON-Schema validators the [Data Model](../architecture/data-model.md) enforces, so generated content is immediately editable in the [Page Builder](page-builder.md).
- **Stay provider-agnostic.** A single `AIProvider` contract abstracts chat/completion, streaming, tool use, and image generation. Drivers exist for Anthropic Claude (default), and the contract is designed so an OpenAI-compatible, Google, or self-hosted driver can be added without touching callers.
- **Be safe by construction.** Every generation passes through input guardrails (prompt-injection screening, capability checks) and output guardrails (schema validation, content-safety review, PII scan) before it is persisted or shown.
- **Be affordable and bounded.** Per-tenant token quotas, Redis rate limits, per-capability cost ceilings, and streaming (to avoid paying for truncated retries) keep spend predictable and multi-tenant-fair.
- **Be extensible.** Plugins register new AI actions (`Ai::action(...)`) that appear in the admin UI and API exactly like built-in capabilities.

---

## 2. Functional Specification

### 2.1 Capabilities

Each capability is a named **AI action** with a typed input schema, a prompt template, an output contract, and a stability tag.

| Capability | Action name | Output artifact | Stability |
|---|---|---|---|
| Website Generator | `website.generate` | `websites` doc + starter `pages`, `menus`, `theme` assignment | `experimental` |
| Landing Page Generator | `page.landing.generate` | one `pages` doc (full Section/Row/Column/Widget tree) | `experimental` |
| Theme Generator | `theme.generate` | `themes` doc + `AssetBundle` (tokens, CSS, layout regions) | `experimental` |
| Widget Generator | `widget.generate` | `widgets` definition + `PropertySchema` + preview template | `experimental` |
| Blog Writer | `blog.write` | `posts` doc (draft) + `taxonomies`/`terms` suggestions | `beta` |
| SEO Optimizer | `seo.optimize` | filtered `page.title`, `meta`, JSON-LD, `redirects` proposals | `beta` |
| Accessibility Checker | `a11y.check` | structured WCAG 2.2 report (issues + fixes), no mutation | `beta` |
| Code Generator | `code.generate` | snippet (Twig-like template, PHP widget stub, CSS) | `experimental` |
| Translation | `content.translate` | localized copy of any content doc, per-field | `beta` |
| Image Generation | `image.generate` | `media` doc (asset written via [Storage](../architecture/storage.md) driver) | `experimental` |
| AI Assistant | `assistant.chat` | streamed conversational turn, optional tool use | `experimental` |

### 2.2 Execution modes

- **Synchronous** (`Ai::run`) — for short, latency-sensitive actions (`seo.optimize` on a single field, `code.generate` for one snippet). Returns the final result inline.
- **Asynchronous + streamed** (`Ai::dispatch`) — the default for structural generation. An `ai_jobs` document is created, an OpenSwoole task worker picks it up, and tokens stream to the browser over SSE (`GET /api/ai/jobs/{id}/stream`) or WebSocket (`/ws/ai`). The final validated artifact is written on completion.
- **Batched** — many small actions (bulk `content.translate` across a `collection`) are grouped into one provider request set where the driver supports it, and results are fanned back out.

### 2.3 Provider abstraction

The platform never calls a vendor SDK directly from a capability. It calls the resolved `AIProvider`. The default provider targets **Anthropic Claude** — `claude-opus-4-8` (Claude Opus 4.8) for structural reasoning and code, `claude-sonnet-5` (Claude Sonnet 5) for high-volume/low-latency actions such as translation and per-field SEO. These models are chosen for capability (long-horizon planning, strict tool/JSON adherence, high instruction fidelity), not because the platform is coupled to them.

---

## 3. Business Requirements

| ID | Requirement | Rationale |
|---|---|---|
| BR-1 | A user with the right capability can produce a publishable draft from a one-paragraph brief in a single action. | Time-to-first-site is the primary adoption metric. |
| BR-2 | No AI feature may be swapped for a different model vendor without a caller code change. | Avoid vendor lock-in; the driver interface is the contract. |
| BR-3 | Every generation is attributable, auditable, and reversible. | Enterprise governance; generated content is drafted, never auto-published. |
| BR-4 | Spend is bounded per workspace and per website with hard ceilings. | Multi-tenant SaaS economics; one tenant cannot exhaust another's budget. |
| BR-5 | Generated output cannot bypass the platform's validation, RBAC, or content-safety rules. | An AI action has exactly the permissions of the invoking user. |
| BR-6 | Credentials for model providers are never stored in the database or exposed to templates. | Secrets live in env/secret manager only. |
| BR-7 | Plugins can add AI actions without core changes. | Ecosystem parity — AI is part of the SDK surface. |

---

## 4. User Stories

- **As a small-business owner**, I describe my bakery in two sentences and get a themed 5-page website draft I can edit in the visual builder, so I never touch code.
- **As a designer**, I generate a theme from a mood description ("warm, editorial, serif display, terracotta accent") and receive design tokens plus layout regions I can refine.
- **As a developer**, I generate a custom widget ("a testimonial carousel with avatar, quote, author, role") and receive a `Widget::register` definition, a `PropertySchema`, and a preview I can drop into `widgets/`.
- **As an author**, I draft a blog post from an outline, then translate it into three languages, all as drafts pending review.
- **As an SEO manager**, I run the optimizer over a page and accept suggested titles, meta descriptions, JSON-LD, and 301 `redirects` for renamed slugs.
- **As an editor**, I run the accessibility checker before publishing and get a prioritized list of WCAG failures with concrete fixes.
- **As a workspace owner**, I set a monthly token quota per website and watch live usage, so spend never surprises me.
- **As a plugin author**, I register an `ai.action` (`translate.legal`) that appears in the admin AI menu alongside built-ins.

---

## 5. Data Model (MongoDB collections & indexes)

The platform owns `ai_jobs` and writes to the shared `notifications`, `audit_logs`, and `media` collections. Every doc carries the standard envelope (`_id`, `created_at`, `updated_at`, `deleted_at`, `version`, `created_by`, `updated_by`) and, being tenant-scoped, `workspace_id` + `website_id`. See the [Data Model](../architecture/data-model.md) reference for the full collection catalog.

### 5.1 `ai_jobs`

```json
{
  "_id": "ObjectId",
  "workspace_id": "ObjectId",
  "website_id": "ObjectId",
  "action": "website.generate",
  "status": "queued",              // queued | running | streaming | review | completed | failed | cancelled
  "provider": "anthropic",
  "model": "claude-opus-4-8",
  "mode": "async",                 // sync | async | batch
  "input": {
    "prompt": "A small artisan bakery in Lyon...",
    "params": { "pages": 5, "locale": "fr-FR" },
    "context_refs": [ { "type": "website", "id": "ObjectId" } ]
  },
  "guardrails": {
    "input_scan": { "verdict": "pass", "signals": [] },
    "output_scan": { "verdict": "pass", "categories": [] }
  },
  "result": {
    "artifact_type": "website",
    "artifact_ids": [ "ObjectId" ],
    "raw_ref": "gridfs://ai/2026/07/…"    // full provider payload, retained per policy
  },
  "usage": {
    "input_tokens": 3120,
    "output_tokens": 8641,
    "cache_read_input_tokens": 2048,
    "cost_micros": 214050,               // integer micro-USD, never float money
    "iterations": 2
  },
  "error": null,                          // { type, message, provider_request_id }
  "idempotency_key": "sha256:…",
  "created_at": "ISODate", "updated_at": "ISODate", "deleted_at": null,
  "version": 3, "created_by": "ObjectId", "updated_by": "ObjectId"
}
```

**Indexes** (declared in `packages/ai`, applied by the migration runner):

| Index | Purpose |
|---|---|
| `{ workspace_id: 1, website_id: 1, created_at: -1 }` | tenant-scoped job history listing |
| `{ status: 1, created_at: 1 }` | task-worker claim scan (queued/running) |
| `{ idempotency_key: 1 }` `unique, partial: {deleted_at: null}` | dedupe retried dispatches |
| `{ workspace_id: 1, "usage.cost_micros": 1, created_at: -1 }` | quota/cost aggregation windows |
| `{ "result.artifact_ids": 1 }` | reverse lookup "what generated this doc" |
| `{ created_at: 1 }` TTL on terminal jobs (policy-configurable) | retention of raw payloads |

A JSON-Schema validator on the collection enforces the `status`/`action`/`provider` enums and the `usage` integer types (cost is always integer micro-USD — money is never stored as a float).

### 5.2 `notifications`

Completion, failure, and quota-threshold events are written to the shared `notifications` collection (`{ recipient_id, type: "ai.job.completed", ref: { job_id }, read_at }`) and pushed live via Redis pub/sub. Cross-collection invariants (write the `pages` tree **and** mark the job `completed` **and** emit the notification) run inside a single [multi-document transaction](../architecture/database-mongodb.md).

---

## 6. Folder Structure

```text
packages/ai/                        # Goco\AI — the platform
├── src/
│   ├── Ai.php                       # Goco\SDK\Ai facade
│   ├── Contracts/
│   │   ├── AIProvider.php           # the driver interface
│   │   ├── StreamSink.php           # abstraction over SSE / WebSocket / Channel
│   │   └── Guardrail.php
│   ├── Providers/
│   │   ├── AnthropicProvider.php    # default — Claude
│   │   ├── ProviderManager.php      # driver resolution (config + Hook filter)
│   │   └── NullProvider.php         # tests / offline
│   ├── Actions/                     # one class per capability
│   │   ├── WebsiteGenerator.php
│   │   ├── LandingPageGenerator.php
│   │   ├── ThemeGenerator.php
│   │   ├── WidgetGenerator.php
│   │   ├── BlogWriter.php
│   │   ├── SeoOptimizer.php
│   │   ├── AccessibilityChecker.php
│   │   ├── CodeGenerator.php
│   │   ├── Translator.php
│   │   ├── ImageGenerator.php
│   │   └── Assistant.php
│   ├── Prompts/
│   │   ├── PromptRegistry.php        # versioned templates
│   │   └── templates/*.md            # {{mustache}} prompt sources
│   ├── Guardrails/
│   │   ├── PromptInjectionScreen.php
│   │   ├── ContentSafetyScreen.php
│   │   └── OutputSchemaValidator.php
│   ├── Jobs/
│   │   ├── GenerationTask.php        # runs inside OpenSwoole task worker
│   │   └── JobRepository.php         # Goco\Database repository over ai_jobs
│   ├── Cost/
│   │   ├── QuotaManager.php          # per-tenant Redis quotas
│   │   └── RateLimiter.php
│   └── Events.php                    # hook/event name constants
├── api/                             # file-based ZealPHP routes
│   └── ai/
│       ├── jobs.php                  # GET /api/ai/jobs  (list)
│       ├── dispatch.php              # POST /api/ai/dispatch
│       └── jobs/{id}/stream.php      # GET  /api/ai/jobs/{id}/stream  (SSE)
├── config/ai.php
└── tests/
```

The developer CLI ships generators for new actions and providers:

```bash
goco make:ai-action translate.legal     # scaffolds an Action + prompt template + test
goco make:ai-provider openai-compatible  # scaffolds an AIProvider driver
```

---

## 7. API Design

All routes are file-based ZealPHP REST endpoints under `apps/api`. They return arrays (auto-JSON) or a Generator (streaming). Auth is JWT for API clients or the Redis session for the admin app; every call is gated by [RBAC](../architecture/permission-system.md) with the `ai.manage` capability plus the target resource capability (e.g. `pages.create` for `page.landing.generate`).

| Method | Path | Capability | Description |
|---|---|---|---|
| `POST` | `/api/ai/dispatch` | `ai.manage` + action's target | Create an `ai_jobs` doc, enqueue, return `{ job_id, stream_url }`. |
| `GET` | `/api/ai/jobs` | `ai.manage` | Tenant-scoped, paginated job history. |
| `GET` | `/api/ai/jobs/{id}` | `ai.manage` | Single job (status, usage, result refs). |
| `GET` | `/api/ai/jobs/{id}/stream` | `ai.manage` | SSE token stream + lifecycle events. |
| `POST` | `/api/ai/jobs/{id}/cancel` | `ai.manage` | Cooperative cancel (sets a Redis flag the worker checks). |
| `POST` | `/api/ai/jobs/{id}/accept` | action's target | Persist a `review`-state artifact as a real draft. |
| `GET` | `/api/ai/usage` | `ai.manage` | Current-window token/cost usage vs quota. |
| `GET` | `/api/ai/actions` | `ai.manage` | Discover available actions (built-in + plugin-registered). |
| `WS` | `/ws/ai` | `ai.manage` | Bidirectional assistant channel (`assistant.chat`). |

### 7.1 Dispatch

```php
// apps/api/api/ai/dispatch.php  →  POST /api/ai/dispatch
use Goco\SDK\Ai;
use Goco\AI\Http\Guard;

return function (array $request, $response) {
    $ctx = Guard::authorize($request, capability: 'ai.manage');

    $job = Ai::dispatch(
        action: $request['action'],           // e.g. 'website.generate'
        input:  $request['input'],
        ctx:    $ctx,                          // carries workspace_id + website_id + user
    );

    $response->status(202);
    return [
        'job_id'     => (string) $job->id,
        'status'     => $job->status,
        'stream_url' => "/api/ai/jobs/{$job->id}/stream",
    ];
};
```

### 7.2 Streaming (SSE)

The stream endpoint returns a Generator; ZealPHP flushes each `yield` as an SSE frame via `$response->sse()`. The worker publishes tokens to a Redis channel; the request coroutine subscribes and relays.

```php
// apps/api/api/ai/jobs/{id}/stream.php  →  GET /api/ai/jobs/{id}/stream
use Goco\SDK\Ai;
use Goco\AI\Http\Guard;

return function (string $id, array $request, $response) {
    Guard::authorize($request, capability: 'ai.manage');
    $response->sse();                          // sets text/event-stream + flush headers

    return (function () use ($id) {
        foreach (Ai::subscribe($id) as $event) {
            // $event: ['type' => 'token'|'status'|'usage'|'done'|'error', ...]
            yield "event: {$event['type']}\n";
            yield 'data: ' . json_encode($event) . "\n\n";
            if ($event['type'] === 'done' || $event['type'] === 'error') {
                return;
            }
        }
    })();
};
```

### 7.3 SDK facade

`Goco\SDK\Ai` is the public entry point. Signatures:

```php
Ai::run(string $action, array $input, Context $ctx): AiResult;      // sync
Ai::dispatch(string $action, array $input, Context $ctx): AiJob;    // async, returns job
Ai::subscribe(string $jobId): Generator;                           // stream events
Ai::action(string $name, array|callable $definition): void;        // register (plugins)
Ai::provider(): AIProvider;                                        // resolve active driver
Ai::usage(Context $ctx, string $window = 'month'): UsageReport;
```

---

## 8. Services

### 8.1 The `AIProvider` driver interface

The single seam that makes the platform vendor-agnostic. A driver implements chat/streaming, tool use, and (optionally) image generation.

```php
namespace Goco\AI\Contracts;

interface AIProvider
{
    public function name(): string;                 // 'anthropic'

    /** Blocking completion — returns the assembled message + usage. */
    public function complete(AiRequest $req): AiResponse;

    /** Streaming — yields token/tool/usage deltas as they arrive. */
    public function stream(AiRequest $req): \Generator;

    /** Image generation; drivers without support throw UnsupportedCapability. */
    public function image(ImageRequest $req): ImageResponse;

    /** Provider-declared model catalog + capability flags. */
    public function models(): array;
}
```

`AiRequest` carries `model`, `system`, `messages`, `tools`, `outputSchema` (for structured output), `thinking`, `effort`, `maxTokens`, and `stream`. The default `AnthropicProvider` maps these onto the Anthropic PHP SDK. Model selection, adaptive thinking, and streaming for large outputs are chosen per the platform's defaults:

```php
namespace Goco\AI\Providers;

use Anthropic\Client;
use Goco\AI\Contracts\AIProvider;

final class AnthropicProvider implements AIProvider
{
    public function __construct(
        private Client $client,   // built from getenv('ANTHROPIC_API_KEY') at boot
    ) {}

    public function name(): string { return 'anthropic'; }

    public function stream(AiRequest $req): \Generator
    {
        // Structural generation streams to avoid HTTP timeouts on 128k outputs,
        // and uses adaptive thinking for planning-heavy actions.
        $stream = $this->client->messages->createStream(
            model:     $req->model ?? 'claude-opus-4-8',
            maxTokens: $req->maxTokens ?? 64000,
            system:    $req->system,
            thinking:  ['type' => 'adaptive'],
            messages:  $req->messages,
            tools:     $req->tools,
            outputConfig: $req->outputSchema
                ? ['format' => ['type' => 'json_schema', 'schema' => $req->outputSchema]]
                : null,
        );

        foreach ($stream as $event) {
            yield $this->normalize($event);   // → GOCO-neutral delta shape
        }
    }
}
```

> **Note**
> The default provider recommends Anthropic Claude — `claude-opus-4-8` (Opus 4.8) for website/theme/widget/code generation and the assistant, and `claude-sonnet-5` (Sonnet 5) for translation and per-field SEO — selected for capability. On the current models adaptive thinking replaces any fixed thinking budget, and non-default sampling parameters are not sent. Swapping to another vendor is a driver change only; capability code never names a model directly — it names a *tier* (`reasoning`, `fast`, `image`) that `ProviderManager` maps to a concrete model.

### 8.2 `ProviderManager`

Resolves the active driver from `config/ai.php` and a Hook filter, so a plugin can substitute a provider per workspace:

```php
$provider = Hook::apply('ai.provider.resolve', $default, $ctx);
```

### 8.3 `GenerationTask`

The unit of asynchronous work, executed inside an OpenSwoole task worker registered at worker start. It: claims the `ai_jobs` doc (status `queued → running`), runs input guardrails, calls `provider->stream()`, publishes each delta to Redis (`Store::publish("ai:{$jobId}", …)`), runs output guardrails on the assembled result, validates against the artifact JSON-Schema, and commits the artifact + job update + notification in one transaction.

```php
// registered once per worker
App::onWorkerStart(function ($server, $wid) {
    App::tick(1000, fn () => JobRepository::reapStale());  // requeue crashed jobs
});
```

Cooperative cancel: the task polls a Redis key `ai:cancel:{jobId}`; the `cancel` endpoint sets it.

---

## 9. Events

Actions and lifecycle transitions are dispatched through the [Event system](../architecture/event-hook-system.md). Action names follow `subject.verb[.tense]`.

| Event | When | Args |
|---|---|---|
| `ai.job.created` | dispatch accepted | `AiJob` |
| `ai.generation.starting` | worker claimed the job | `AiJob` |
| `ai.generation.streaming` | first token emitted | `AiJob, delta` |
| `ai.generation.completed` | artifact persisted | `AiJob, artifactIds` |
| `ai.generation.failed` | provider or guardrail error | `AiJob, error` |
| `ai.guardrail.blocked` | input or output screen rejected | `AiJob, verdict` |
| `ai.quota.threshold` | tenant crossed 80% / 100% of quota | `Context, UsageReport` |
| `ai.artifact.accepted` | user promoted a `review` artifact to draft | `AiJob, artifact` |

`ai.generation.completed` is dispatched **asynchronously** (`Hook::dispatchAsync`) so post-generation side effects (indexing into [Search](../architecture/search.md), warming caches) never block the response.

---

## 10. Hooks

Filters (`subject.noun`) let plugins and themes shape prompts, model choice, and output.

| Filter | Purpose | Value |
|---|---|---|
| `ai.provider.resolve` | choose the driver per request | `AIProvider` |
| `ai.model.tier` | map an abstract tier to a concrete model | `string` model id |
| `ai.prompt.template` | override or wrap a prompt template | rendered prompt string |
| `ai.request.params` | inject `system`, tools, `maxTokens` per action | `AiRequest` |
| `ai.output.artifact` | transform the parsed artifact before validation | artifact array |
| `ai.guardrail.policies` | add/modify guardrails for a tenant | `Guardrail[]` |
| `ai.cost.quota` | resolve the effective quota for a tenant | `Quota` |

Plugin-registered actions namespace their hooks by slug (`legal-tools:ai.prompt.template`). Registering a new action:

```php
use Goco\SDK\Ai;
use Goco\SDK\Plugin;

Plugin::register('legal-tools', [/* manifest */]);
Plugin::boot('legal-tools');

Ai::action('translate.legal', [
    'tier'        => 'reasoning',
    'input'       => ['text' => 'string', 'jurisdiction' => 'string'],
    'template'    => 'legal-tools::translate-legal',
    'output'      => TranslateLegalSchema::class,     // JSON-Schema for validation
    'capability'  => 'content.translate',
]);
```

---

## 11. UI Architecture

The AI experience lives in `apps/admin` and is composable into any screen:

- **Command palette / brief bar.** A single input on the dashboard ("Describe your website…") that dispatches `website.generate` and opens the streaming view.
- **Streaming canvas.** Subscribes to the SSE stream; renders the artifact incrementally. For structural actions the partial Section/Row/Column tree renders live in a preview so the user watches the site assemble.
- **Assistant panel.** A persistent WebSocket-backed (`/ws/ai`) chat that can call tools (create page, edit widget) with confirmation gates before any mutating tool runs.
- **Review + accept.** Generated artifacts land in `review` status; the user diffs and accepts, which promotes them to a draft in the [Page Builder](page-builder.md) or [Blog Engine](blog-engine.md). Nothing auto-publishes.
- **Usage meter.** Live quota/cost readout fed by `GET /api/ai/usage`, with a warning banner at the `ai.quota.threshold` event.

Admin fragments are rendered with ZealPHP `App::fragment()` (htmx regions), so the streaming canvas updates a single region without a full page reload.

---

## 12. Security Model

The AI Platform is treated as an untrusted-input, untrusted-output boundary.

- **RBAC everywhere.** An AI action runs with **exactly** the invoking user's capabilities — never more. `page.landing.generate` requires `pages.create`; `image.generate` requires `media.upload`. Accepting a generated artifact is a second, separately-authorized write. See the [Permission System](../architecture/permission-system.md) and [Security Model](../security/security-model.md).
- **Prompt-injection defense.** All user- and content-derived text is screened by `PromptInjectionScreen` before it enters a prompt. Retrieved GOCO content injected as context is passed as *data*, clearly delimited, never as instructions. The provider is instructed to treat page/website context as reference, and tool-use calls that mutate data always require an explicit user confirmation in the UI — the model can *propose* a mutation but cannot *commit* one.
- **Output review.** Generated structure is validated against the same JSON-Schema validators MongoDB enforces; anything that fails is rejected, never persisted. Generated prose passes `ContentSafetyScreen` (a provider-side safety pass plus a category filter). Generated code is never executed server-side — `code.generate` returns text into the editor only.
- **Credential handling.** Provider API keys come from environment/secret manager (`ANTHROPIC_API_KEY`, injected via the [Docker](../deployment/docker.md) service env and Traefik-terminated TLS). Keys are **never** written to MongoDB, never returned by any endpoint, and never exposed to templates or the `\ZealPHP\G` per-request context. Redacted request IDs are logged for support; raw payloads are stored in GridFS under a retention policy and are readable only with `ai.manage`.
- **CSRF + auth.** Dispatch and accept are state-changing and pass through the ZealPHP `Csrf` middleware; the assistant WebSocket authenticates the session on `onOpen` and rejects unauthenticated upgrades.
- **Audit.** Every job writes to `audit_logs` (actor, action, tenant, model, token usage, guardrail verdicts). Generated artifacts carry `created_by` = the invoking user, not a system principal.
- **Tenant isolation.** `context_refs` and retrieved context are always filtered by `workspace_id` + `website_id`; a job can never pull another tenant's content into a prompt. See [Multi-Tenancy](../architecture/multi-tenancy.md).

---

## 13. Performance Strategy

- **Streaming first.** Structural actions stream token-by-token, so the UI shows progress immediately and the platform never pays for a truncated non-streaming retry. Large outputs (up to 128k tokens on the default models) require streaming to avoid HTTP timeouts.
- **Task workers, not the request loop.** Generation runs in OpenSwoole task workers; request coroutines only relay the Redis stream, so a slow multi-minute generation never blocks a worker's HTTP loop.
- **Prompt caching.** Stable prompt prefixes (system prompt, tool definitions, the GOCO domain schema injected into every structural prompt) are marked cacheable so repeated dispatches read the cache at a fraction of the input cost. The platform keeps that prefix byte-stable — no timestamps or per-request IDs ahead of the cache breakpoint. Cache hit rate is surfaced in `usage.cache_read_input_tokens`.
- **Tiering.** `fast`-tier actions (`content.translate`, per-field `seo.optimize`) route to Claude Sonnet 5; `reasoning`-tier actions route to Claude Opus 4.8. Effort is tuned per action rather than defaulting to maximum.
- **Batching.** Bulk translation groups items into batched provider requests, sharing the cached instruction prefix across all items.
- **Redis for hot state.** Live token relay, cancel flags, rate-limit counters, and quota windows are all Redis — no MongoDB write is on the token-streaming hot path. Durable state (`ai_jobs`, usage rollups) is written on lifecycle transitions only.
- **Backpressure.** A `ConcurrencyLimit` middleware and a Redis semaphore cap concurrent generations per worker and per tenant so one workspace's burst cannot starve others.

---

## 14. Testing Strategy

- **Provider contract tests.** A shared conformance suite runs every `AIProvider` driver (Anthropic + `NullProvider`) against the same expectations (streaming order, usage accounting, tool round-trip, `UnsupportedCapability` behavior).
- **Deterministic capability tests.** Actions are tested against the `NullProvider`, which replays recorded fixtures, so capability logic (prompt assembly, output parsing, schema validation, artifact persistence) is tested with zero network calls and zero token spend.
- **Guardrail tests.** A red-team fixture set of prompt-injection and unsafe-content inputs asserts `ai.guardrail.blocked` fires and nothing is persisted.
- **Schema-validation tests.** Generated-artifact fixtures are validated against the live MongoDB JSON-Schema validators to guarantee generated output is always editable in the builder.
- **Integration tests.** A gated suite runs a small number of real Anthropic calls (behind a CI secret) to catch provider-API drift; skipped when `ANTHROPIC_API_KEY` is absent.
- **Cost/quota tests.** Simulated usage asserts rate limits, quota thresholds, and the `ai.quota.threshold` event fire at the right boundaries.

See the project-wide [Testing Strategy](../community/testing-strategy.md).

---

## 15. Extension Points

- **New provider** — implement `AIProvider`, register via `ProviderManager` or the `ai.provider.resolve` filter. `goco make:ai-provider` scaffolds it.
- **New capability** — `Ai::action(...)` from a plugin; it appears in the admin AI menu, the `GET /api/ai/actions` catalog, and the CLI.
- **Prompt overrides** — the `ai.prompt.template` filter and the versioned `PromptRegistry` let themes/plugins tune generation without forking core.
- **Custom guardrails** — implement `Guardrail`, add it via `ai.guardrail.policies` (e.g. a tenant-specific brand-safety screen).
- **Model routing** — the `ai.model.tier` filter remaps tiers to concrete models per workspace (e.g. an enterprise tenant pinning a specific model version).
- **Tool integration** — actions declare tools that the provider can call; a plugin can register tools the assistant may use (behind confirmation gates).

---

## 16. Upgrade Strategy

- **Semantic Versioning.** `gococms/ai` follows [SemVer](../roadmap.md); while `experimental`, minor releases may include breaking changes to the `AIProvider` contract and the `ai_jobs` schema — each is called out in the [Changelog](../changelog.md).
- **Prompt versioning.** Prompt templates are versioned in `PromptRegistry`; a job records the template version it used, so historical jobs remain reproducible and prompt upgrades never silently change behavior mid-flight.
- **Model migration.** Because capability code names *tiers*, not model ids, a model version bump is a one-line change in `config/ai.php` (or the `ai.model.tier` filter). Provider-side model migrations (parameter deprecations, tokenizer shifts) are absorbed inside the driver, not in callers.
- **Schema migrations.** `ai_jobs` changes ship with migrations in `packages/database`; the `raw_ref` payload format is forward-compatible (additive fields only within a major).
- **Deprecation policy.** A removed capability or renamed action name is aliased for one minor cycle and emits a deprecation notice before removal.

---

## 17. Future Roadmap

- **Additional first-class drivers** — OpenAI-compatible, Google, and self-hosted/Open-weights drivers implementing the same `AIProvider` contract.
- **Retrieval-grounded generation** — first-class RAG over a website's own content and [collections](database-builder.md), using the [Search](../architecture/search.md) provider as the retriever, so generated copy cites and stays consistent with existing pages.
- **Managed multi-step agents** — long-horizon "build my whole site" runs with server-side planning, checkpointing to `ai_jobs`, and per-step review gates.
- **Live translation sync** — keep localized content in lockstep with source edits via change-driven `content.translate` jobs.
- **Fine-grained brand controls** — per-workspace style/voice profiles injected as reusable, cacheable prompt prefixes.
- **Evaluation harness** — automatic scoring of generated artifacts (accessibility score, SEO score, schema-conformance) surfaced before accept.
- **On-device/edge image models** — a local `image` driver for privacy-sensitive deployments.

---

## Related

- [Widget Engine](widget-engine.md) — the target of `widget.generate`.
- [Theme Engine](theme-engine.md) — the target of `theme.generate`.
- [Page Builder](page-builder.md) — where generated pages are reviewed and edited.
- [Blog Engine](blog-engine.md) — the target of `blog.write`.
- [Database Builder](database-builder.md) — dynamic collections used for RAG grounding.
- [Widget SDK](../sdk/widget-sdk.md) · [Plugin SDK](../sdk/plugin-sdk.md) · [Hook SDK](../sdk/hook-sdk.md) · [CLI SDK](../sdk/cli.md)
- [Event & Hook System](../architecture/event-hook-system.md) · [Caching, Queue & Realtime](../architecture/caching-and-queue.md) · [Storage & Media](../architecture/storage.md) · [Search](../architecture/search.md)
- [MongoDB Data Layer](../architecture/database-mongodb.md) · [Data Model](../architecture/data-model.md) · [Multi-Tenancy](../architecture/multi-tenancy.md)
- [Permission System](../architecture/permission-system.md) · [Security Model](../security/security-model.md)
- [Docker Architecture](../deployment/docker.md) · [Configuration](../getting-started/configuration.md)
- [Documentation Index](../README.md)
