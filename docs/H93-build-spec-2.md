# H93 build spec — round 2 (recommendations 1–6 + fixes)

Repository `/home/user/storefront-`, branch `h93/phase1-review-signals` (HEAD has phases 1–5 of round 1). Same
conventions and checks as `H93-build-spec.md`. Local Postgres 16 on 127.0.0.1:5433 has **pgvector 0.6** installed
(`create extension vector` works); Supabase has it too.

Checks that must pass before a stage is done:
```
cd /home/user/storefront- && pnpm format:check && pnpm typecheck && TEST_DATABASE_URL=postgres://postgres@127.0.0.1:5433/lens_test pnpm --filter @lens/core test && pnpm build
```

## Migration `db/migrations/0006_h93_round2.sql` (DDL only)

```sql
-- pgvector is optional: everything below still works without it, embeddings just stay disabled.
do $$ begin
  if exists (select 1 from pg_available_extensions where name = 'vector') then
    execute 'create extension if not exists vector';
    execute 'create table if not exists label_embeddings (label text primary key, model text not null, v vector(1024) not null, created_at timestamptz not null default now())';
    execute 'alter table opportunities add column if not exists centroid vector(1024)';
    execute 'create index if not exists opportunities_centroid on opportunities using hnsw (centroid vector_cosine_ops)';
  end if;
end $$;

-- ground truth for the classifier
create table if not exists review_verdicts (
  id bigserial primary key,
  source text not null check (source in ('review','item')),
  ref text not null,                       -- review: '<app_id>:<review_id>', item: '<id>' (same as Evidence.ref)
  verdict text not null check (verdict in ('correct','wrong')),
  corrected jsonb,                         -- {"wtp_signal":"...","label_kind":"...","pain_score":3,"label":"..."} any subset
  notes text,
  analysis_version smallint not null,
  created_at timestamptz not null default now(),
  unique (source, ref)
);

-- Anthropic Message Batches in flight
create table if not exists ai_batches (
  id bigserial primary key,
  provider_batch_id text not null unique,
  kind text not null check (kind in ('reviews','items')),
  app_id uuid references apps(id) on delete cascade,
  status text not null default 'submitted' check (status in ('submitted','ended','failed')),
  request_count int not null default 0,
  payload jsonb not null,                  -- {"requests":[{"custom_id":"...","ids":["review_id",...]}]}
  error text,
  created_at timestamptz not null default now(),
  ended_at timestamptz
);
create index if not exists ai_batches_open on ai_batches (status) where status = 'submitted';

alter table opportunities
  add column if not exists validation jsonb,          -- generated kit, see generateValidation
  add column if not exists validation_metrics jsonb;  -- {"waitlist":0,"price_clicks":0,"replies":0,"started_at":"...","recorded_at":"..."}

-- RLS like the others
alter table review_verdicts enable row level security;
alter table ai_batches enable row level security;
do $$ begin if to_regclass('label_embeddings') is not null then execute 'alter table label_embeddings enable row level security'; end if; end $$;
```

## Settings (`packages/core/src/settings.ts`)

```ts
ai: {
  provider: "openai" | "anthropic"      // default "openai" (existing OpenAI-compatible path)
  baseUrl, apiKey, model, autoAnalyse    // unchanged; for anthropic, baseUrl may be empty (SDK default) — keep it for tests
  batch: boolean                         // anthropic only; default false; classification goes through Message Batches (50% off)
  embedding: { baseUrl: string; apiKey: string; model: string; dimensions: number }   // all empty = disabled; dimensions default 1024
}
score: { listingWeight: 0.5, painWeight: 1, wtpWeight: 2, halfLifeDays: 90, autoMapThreshold: 0.86 }
reddit: { enabled: false, clientId: "", clientSecret: "", userAgent: "", subreddits: string[], keywords: string[], limit: 50, apiBase: "" /* test override */ }
```
`getSettings()` returns all keys merged with defaults; `saveSettings(key, patch)` works for the new keys.

## Stage E1 — AI providers, embeddings, score weights, verdicts (core only)

### `ai.ts`
- Add dependency `@anthropic-ai/sdk` to `packages/core/package.json` (latest). Build the client with `new Anthropic({ apiKey, baseURL: cfg.baseUrl || undefined })`.
- `chat(cfg, messages, opts)` dispatches on `cfg.provider`:
  - `openai` → existing code, unchanged.
  - `anthropic` → `client.messages.create({ model, max_tokens, system: [{ type: "text", text: systemText, cache_control: { type: "ephemeral" } }], messages: [{ role: "user", content }] })`, temperature 0.2 omitted (not needed). When `opts.json` is set, append "Respond with JSON only." to the user turn (structured outputs are not used: replies are parsed by `parseJsonReply`). Return the concatenated `text` blocks. Check `stop_reason === "refusal"` → throw `AiError("The model declined this request")`. Wrap SDK errors into `AiError` with the same friendly hints as the OpenAI path (401/403 key, 404 model, network). Note in a comment: Haiku 4.5 needs a ≥4096-token prefix to cache, Sonnet 5.5 / Opus 5.5 need 512 — so caching mostly benefits the larger models here.
- `testConnection` works for both providers.
- `embed(cfg.embedding, texts: string[], opts?: { inputType?: "query" | "document" })` → POST `${baseUrl}/embeddings` with `{ model, input: texts, dimensions }` (OpenAI-shaped; Voyage's `/v1/embeddings` accepts the same plus optional `input_type` — send it only when `baseUrl` contains "voyageai"). Returns `number[][]`. `embeddingConfigured(settings)` helper. Chunk inputs by 100.
- `batchClient` helpers for Anthropic Message Batches: `createBatch(cfg, requests: {custom_id, system, user, maxTokens}[])` → provider batch id; `retrieveBatch(cfg, id)` → `{ status: "in_progress" | "ended", counts }`; `batchResults(cfg, id)` → async iterable of `{ custom_id, ok: boolean, text?: string, error?: string }` using `client.messages.batches.results(id)` (result.type succeeded → text; errored/expired/canceled → error).

### `analysis.ts`
- Split `analyseApp` into `classifyApp(appId, opts)` (classification only; returns `{classified, pending}`) and `summariseApp(appId, opts)` (the sentiment/labels/insights part). `analyseApp` = classify then summarise (unchanged behaviour), **except** when `settings.ai.provider === "anthropic" && settings.ai.batch`: then it calls `submitClassificationBatch("reviews", appId)` and returns `{ classified: 0, reviewsCount: 0, batched: n }` without summarising (the batch poller summarises when results land). Same for `analyseItems` (`kind: "items"`, no summarise; the poller enqueues `group_labels`).
- `submitClassificationBatch(kind, appId?)`: builds the same per-batch payloads as `classifyBatch` (20 per request), creates the provider batch, inserts `ai_batches` with `{requests:[{custom_id, ids}]}`. If there is nothing pending, returns 0 without calling the provider.
- `pollBatches(opts)`: for each `ai_batches` row with status submitted: retrieve; if ended → for each result, look up the ids, re-read their texts from the DB (reviews or items), `normaliseItem(item, sourceText)` and apply the same update as `classifyBatch` (extract a shared `applyClassifications(kind, appId, items)` helper). Mark the row ended (or failed with the error, keeping a retry possible by leaving `analysed_at` null). After a reviews batch ends → `summariseApp(appId)`; after an items batch → `enqueue("group_labels")`. Returns `{ polled, ended }`.

### `opportunities.ts`
- `statsSelect` reads `settings.score` and uses the weights as SQL parameters: `recent = sum(exp(-ln(2)/halfLifeDays * age_days))`, `score = recent * (1 + listingWeight*(listings-1)) * (1 + painWeight*avg_pain/5) * (1 + wtpWeight*wtp_share)`. `opportunityScore()` takes the same weights (default = defaults). `listOpportunities`/`getOpportunity` fetch settings once.
- Verdicts: `recordVerdict({source, ref, verdict, corrected?, notes?})` (upsert; `analysis_version` from the evidence row; when `corrected` has fields, also write them onto the review/item row so the fix is visible everywhere), `deleteVerdict(source, ref)`, `reviewQueue({limit = 20, onlySignals?: boolean})` → Evidence rows analysed with the current version and without a verdict, ordered: signals ≠ none first, then random (`order by (wtp_signal <> 'none') desc, random()`), `accuracyStats()` → `{ total, correct, wrong, accuracy, byVersion: [{version, total, correct}], bySignal: [{signal, total, correct}], byKind: [...] }` (computed over verdicts joined to evidence; a verdict is "correct" when verdict='correct').
- Embeddings (grouping): `embedLabels()` embeds every label in `opportunity_labels` ∪ unmapped labels that has no row in `label_embeddings` (skips silently when embeddings are not configured or `label_embeddings` does not exist — check `to_regclass`). `recomputeCentroids(ids?)` sets `opportunities.centroid = avg(v)` over member labels (use `avg()` on vector — pgvector supports it). In `groupLabels`, before calling the model: if embeddings configured and the table exists, embed the unmapped labels, run one query that finds for each label the nearest opportunity centroid (`1 - (v <=> centroid)`) and auto-map those with similarity ≥ `score.autoMapThreshold` (killed opportunities included — that is the graveyard); only the remainder goes to the model; afterwards recompute centroids for touched opportunities. `similarOpportunities(id, limit = 5)` → other opportunities by centroid similarity with `status` (used by the UI to show "similar / previously killed").

### Tests
- `ai.test.ts`: anthropic provider against a fake HTTP server implementing `POST /v1/messages` (returns `{content:[{type:"text",text:...}], stop_reason:"end_turn", usage:{}}`) — verify system cache_control is sent and text is returned; refusal → AiError; `embed()` against a fake `/embeddings`.
- `batches.test.ts` (integration, needs DB): fake `POST /v1/messages/batches`, `GET /v1/messages/batches/:id` (first in_progress, then ended), `GET /v1/messages/batches/:id/results` (JSONL) → `analyseApp` in batch mode submits, `pollBatches` applies classifications and writes insights; items batch enqueues `group_labels`.
- `integration.test.ts`: score weights change the score (set `listingWeight` to 0 → score drops for a multi-listing opportunity); verdicts: `reviewQueue` excludes recorded ones, `accuracyStats` counts, `corrected` writes through; embeddings: with a fake `/embeddings` server returning deterministic vectors (e.g. hash of the text into 1024 dims, identical text → identical vector), `embedLabels` fills `label_embeddings`, `recomputeCentroids` sets centroids, `groupLabels` auto-maps a label whose vector matches an existing (killed) opportunity without calling the chat model (assert the fake chat endpoint was not hit), `similarOpportunities` returns it.

## Stage E2 — Reddit source, validation kit (core only)

### `reddit.ts` (new)
- `fetchReddit(opts)`: reads `settings.reddit`; if disabled or missing credentials → return `{ fetched: 0, inserted: 0, skipped: 0 }`. OAuth app-only token: `POST {authBase}/api/v1/access_token` with basic auth (clientId:clientSecret), body `grant_type=client_credentials`, header `User-Agent: userAgent` (required by Reddit). Then for each subreddit and each keyword: `GET {apiBase}/r/{sub}/search?q={keyword}&restrict_sr=1&sort=new&t=month&limit={limit}` with the bearer token and User-Agent; collect posts (`data.children[].data`: `title`, `selftext`, `permalink`, `author`, `created_utc`, `id`). Body = `title + "\n\n" + selftext` (skip empty selftext-only link posts with no text). Sleep 700 ms between requests (stay far under 100/min). Insert via `importItems` with `source: "reddit"`, `url: "https://www.reddit.com" + permalink`, `author`, `postedAt`. On success with inserted > 0 → `enqueue("analyse_items")`. `apiBase` default `https://oauth.reddit.com`, `authBase` default `https://www.reddit.com`; `settings.reddit.apiBase` overrides both for tests (auth at `${apiBase}/api/v1/access_token`).
- Log every request count; surface a clear error when the token call returns 401/403 ("Reddit refused the credentials — check the app's client id/secret and that the app type is 'script'").
- Add a comment block on the Responsible Builder Policy: personal, non-commercial use only unless approved; do not raise the rate.

### `opportunities.ts` — validation kit
- `generateValidation(id, opts)`: builds a prompt from `getOpportunity` (label, top 8 quotes, competitors, price hints from `competitor_mentioned` + any "$" amounts in evidence text) and asks the model (`json: true`) for:
  `{"headline":"…","subheadline":"…","bullets":["…","…","…"],"cta":"Start 7-day trial — $X/mo","price":"$X/mo","thread_reply":"…","waitlist_copy":"…"}`.
  Rules in the prompt: bullets must paraphrase real quotes (no invented claims), thread_reply is a public, non-spammy reply for the original thread that asks if they want to try a rough prototype, price copied from a named competitor when known else "$4.99/mo". Validate/truncate, store `validation = {...fields, generated_at}`.
- `saveValidationMetrics(id, {waitlist, trial_clicks: price_clicks, replies, started_at?})` → stores `validation_metrics` with `recorded_at`; `validationDecision(metrics)` (pure, exported): `proceed` if `waitlist >= 20 || price_clicks >= 5 || replies >= 3`; `kill` if none of those and `started_at` is ≥ 5 days ago; else `pending`. Returned alongside in `getOpportunity` as `validation_decision`.

### Tests
- `reddit.test.ts`: fake server for token + search → items inserted with source reddit and url; second run inserts 0 (dedupe); disabled settings → no HTTP calls; 401 → friendly error.
- integration: `generateValidation` stores the kit (fake AI answers the validation prompt — detect by "headline" in system), `validationDecision` cases, `saveValidationMetrics` round trip.

## Stage F — worker + MCP
- Worker: `poll_batches` runs from the 60 s interval when `ai.provider === "anthropic" && ai.batch` (call `pollBatches` directly, guarded by a module-level "in flight" flag); jobs `fetch_reddit` → `fetchReddit`, `generate_validation` ({opportunityId}) → `generateValidation`, `embed_labels` → `embedLabels` + `recomputeCentroids`. Nightly (after `sync_all`): enqueue `fetch_reddit` when enabled, then `group_labels`. Add all to `JobType`.
- `activeJobs()` / `/api/jobs` also return `opportunity_id` (`payload->>'opportunityId'`) so the UI can scope busy states.
- MCP tools (total 29): `record_verdict` {source, ref, verdict, corrected?, notes?}, `get_accuracy` (readOnly), `get_review_queue` {limit?, only_signals?} (readOnly), `get_validation` {opportunity_id} (readOnly; kit + metrics + decision), `save_validation_metrics` {opportunity_id, waitlist?, price_clicks?, replies?, started_at?}, `similar_opportunities` {opportunity_id} (readOnly; empty when embeddings are off). Update smoke test count and add calls; update `TOOLS` in settings page and README table.

## Stage G — web
- `/review`: "Label review" page: queue of 20 evidence cards (app, rating, body with evidence highlighted, the model's label / kind / signal / pain) each with Correct / Wrong buttons; Wrong opens a small inline form to correct signal/kind/pain/label (optional) + notes; toggle "Only items with signals"; header stat tiles from `accuracyStats` (overall accuracy, verdicts count, per-signal accuracy table). Sidebar + command menu entry "Review" with count of verdicts.
- Settings: AI card gets a Provider select (OpenAI-compatible | Anthropic) — when Anthropic: base URL optional, model placeholder `claude-haiku-4-5`, a "Use Message Batches (50% cheaper, results within ~1h)" switch; new "Embeddings" card (base URL, key, model, dimensions, Test button that embeds "hello" and shows the dimension count, and a note about Voyage/OpenAI); new "Scoring" card (the 5 weights with descriptions, Save); new "Reddit" card (enabled, client id, secret (masked like the AI key), user agent, subreddits textarea one per line, keywords textarea one per line, limit, "Fetch now" button → enqueue `fetch_reddit`, and a short compliance note).
- Opportunity page: new **Validation** tab: Generate kit button (→ `generate_validation` job, JobWatcher scoped by opportunity_id), shows headline/subheadline/bullets/CTA/price with Copy buttons, thread reply with Copy, waitlist copy; metrics form (waitlist, price clicks, replies, started at) + Save; decision badge (proceed / kill suggested / pending) with the rule text. Header: "Similar opportunities" chips (from `similarOpportunities`, killed ones marked) when embeddings are on. Spec tab and Validation tab busy state scoped by `opportunity_id`.
- `/opportunities` page: add an **Outcomes** view (tab or toggle): shipped + killed opportunities with score, gate pass count, validation decision, outcome numbers (installs / trial rate / paying per 100), killed reason — the calibration table.
- Fixes: `not-found.tsx` text → "This page may have been removed." with Back to library; nothing app/board-specific.
- Dashboard: "Review accuracy" tile linking to `/review`.

## Stage H — docs
- README (Arabic + English): providers (Anthropic + batches + caching note), embeddings, scoring weights, Reddit (setup: create a "script" app at reddit.com/prefs/apps, policy note), label review + accuracy, validation kit and decision rule, outcomes view, new tools.
- CLAUDE.md: provider dispatch lives in ai.ts; batch classification is asynchronous (ai_batches + pollBatches); embeddings optional (guarded by to_regclass).
