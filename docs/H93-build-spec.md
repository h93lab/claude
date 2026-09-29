# H93 build spec (phases 2–5) — for implementation agents

Repository: `/home/user/storefront-` (pnpm monorepo, Node 22). Branch `h93/phase1-review-signals` already contains phase 1
(migration 0004: per-review `wtp_signal`, `competitor_mentioned`, `workaround`, `evidence_span`, `pain_score`, `raw_analysis`,
`analysis_version`; `analyse_all` job; `queueReanalysis()`; `signalCounts()`; `getReviews({signal})`).

Read `CLAUDE.md` and `README.md` first and follow their conventions exactly (batched SQL, `prepare: false`, migrations are
numbered and DDL-only, server actions enqueue long work, client components never import runtime values from `@lens/core`).

Checks that must pass before a stage is done:
```
pnpm format:check && pnpm typecheck && TEST_DATABASE_URL=postgres://postgres@127.0.0.1:5433/lens_test pnpm --filter @lens/core test && pnpm build
```
A local Postgres 16 runs on 127.0.0.1:5433 (user `postgres`, no password). Database `lens` is seeded for the MCP smoke test:
```
export DATABASE_URL=postgres://postgres@127.0.0.1:5433/lens MEDIA_DIR=/tmp/lenspg/media MCP_TOKEN=dev-token-0123456789abcdefgh PUBLIC_URL=http://localhost:3000
pnpm migrate && (pnpm dev:mcp > /tmp/lenspg/mcp.log 2>&1 &) && sleep 6 && node apps/mcp/test/smoke.mjs http://localhost:3001/mcp $MCP_TOKEN; pkill -f apps/mcp
```

## Product model

An **opportunity** is one recurring missing capability across the tracked apps ("Offline mode — sleep stories"), with the
reviews (and imported items) that evidence it. Opportunities are formed by grouping the AI `label`s of complaint/request
reviews across all apps — no embeddings. Each opportunity moves through `surfaced → validating → building → shipped → killed`.
Killed opportunities are the graveyard: new labels that mean the same thing map onto them and stay hidden.

## Migration `db/migrations/0005_opportunities.sql` (DDL only)

```sql
alter table apps add column if not exists own boolean not null default false;   -- the user's own shipped app

create table if not exists opportunities (
  id bigserial primary key,
  label text not null,                        -- canonical English label "<missing capability> — <context>"
  kind text not null default 'complaint' check (kind in ('complaint','request')),
  status text not null default 'surfaced' check (status in ('surfaced','validating','building','shipped','killed')),
  notes text,
  gate jsonb,                                 -- {"checks":{"scope":true,...},"notes":"...","checked_at":"..."}
  spec_md text,
  spec_generated_at timestamptz,
  outcome jsonb,                              -- {"installs":0,"trial_starts":0,"paying":0,"notes":"","recorded_at":"..."}
  killed_reason text,
  revisit_after timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create index if not exists opportunities_status on opportunities (status, updated_at desc);

-- every distinct lowercased review label maps to exactly one opportunity
create table if not exists opportunity_labels (
  label text primary key,                     -- lower(trim(label))
  opportunity_id bigint not null references opportunities(id) on delete cascade,
  created_at timestamptz not null default now()
);
create index if not exists opportunity_labels_opp on opportunity_labels (opportunity_id);

-- text from outside the stores: pasted posts, support emails, forum threads
create table if not exists items (
  id bigserial primary key,
  source text not null default 'paste',       -- paste | reddit | support | other
  url text,
  author text,
  body text not null,
  app_id uuid references apps(id) on delete set null,
  posted_at timestamptz,
  fetched_at timestamptz not null default now(),
  content_hash text not null,                 -- md5(lower(regexp_replace(body,'\s+',' ','g')))
  sentiment text, topic text, label text, label_kind text,
  wtp_signal text, competitor_mentioned text, workaround text, evidence_span text, pain_score smallint,
  raw_analysis jsonb, analysis_version smallint not null default 0, analysed_at timestamptz,
  unique (source, content_hash)
);
create index if not exists items_pending on items (analysed_at) where analysed_at is null;
-- RLS like the other tables (see 0001): enable on opportunities, opportunity_labels, items
```

## Core (`packages/core/src`)

### `opportunities.ts` (new)
- `groupLabels(opts)` — the grouping job. Steps:
  1. Collect distinct `lower(trim(label))` with counts from `reviews` (label_kind in complaint/request) **union** `items`,
     that have no row in `opportunity_labels`. If none, return `{ mapped: 0, created: 0 }`.
  2. Load existing opportunities (id, label, kind, status) — including killed ones.
  3. In batches of 120 labels, ask the configured model (reuse `chat()` from `ai.ts`, `json: true`) with a system prompt:
     given existing opportunities `[{id,label}]` and new labels `[{label,count,kind}]`, return
     `{"map":[{"label":"...","opportunity_id":123}] , "new":[{"label":"canonical label","kind":"complaint|request","labels":["...","..."]}]}`.
     Rules in the prompt: map only when the underlying missing capability is the same; a new opportunity's canonical label
     is the clearest of its member labels, ≤60 chars, form "<missing capability> — <context>", no app names; every input
     label must appear exactly once in either `map` or a `new` group; killed opportunities count as existing (so duplicates
     of killed ideas map to them and stay hidden).
  4. Validate: unknown ids or labels not in the batch are dropped; labels the model forgot are grouped as their own new
     opportunity (label = the label itself). Insert new opportunities and all `opportunity_labels` rows in batched
     statements (`sql(rows)` / unnest). `on conflict (label) do nothing`.
  5. Honour `signal` (AbortSignal), `log`, `onProgress` like `analyseApp`.
- `opportunityStats()` / `listOpportunities({status?, kind?, own?: 'exclude'|'only'|'all', q?, sort?})` — one SQL query
  over the **evidence union** (reviews joined to apps + items) grouped by opportunity, returning per opportunity:
  `n` (evidence count), `listings` (distinct `(store, store_id)` — NOT app_id, so the same app in several countries counts once),
  `apps` (distinct app_id), `avg_pain`, `neg_share` (share of evidence with rating ≤ 2, items count as 1 when pain ≥ 3),
  `signals` (`{paying_competitor, churned, workaround, stated_wtp}` counts), `recent` (sum of exp(-ln(2)/90 * age_days) over
  evidence dates, i.e. recency-weighted count with a 90-day half-life; missing dates count as 180 days old), `last_seen`,
  `competitors` (top 5 competitor_mentioned with counts), and the **score**:
  `score = recent * (1 + 0.5 * (listings - 1)) * (1 + avg_pain / 5) * (1 + 2 * wtp_share)` where
  `wtp_share = (signals except none) / n`. Store nothing; compute in the query (it's fine at this scale; add indexes on
  `reviews (label)` and `items (label)` in the migration if missing). `own='exclude'` (default) drops evidence from apps
  with `own = true`; `own='only'` keeps only it (this is the "my apps" view).
- `getOpportunity(id)` — the row + stats + `evidence` (newest first, limit 200): each with `source` ('review'|'item'),
  app name/icon/store/country, rating, title, body, evidence_span, wtp_signal, competitor_mentioned, workaround, pain_score,
  date, url (store url or item url), plus `labels` (member labels with counts).
- `searchEvidence({q?, signal?, label?, minPain?, own?, limit, offset})` — cross-app evidence search for MCP.
- `setOpportunityStatus(id, status, {reason?, revisitAfter?})`, `saveGate(id, gate)`, `saveSpec(id, md)`,
  `recordOutcome(id, outcome)`, `updateOpportunity(id, {label?, notes?, kind?})`, `mergeOpportunities(fromId, intoId)`
  (repoints labels, deletes `from`). All bump `updated_at`.
- `generateSpec(id, {signal, log, onProgress})` — builds a prompt from `getOpportunity` (top 12 evidence quotes with app,
  rating, date; competitors; signals; listings) and asks the model for a Markdown SPEC with exactly these sections:
  1 Problem in the users' words (quotes with `Q1..Qn` ids, app, rating, date), 2 Target user and trigger moment,
  3 MVP scope: exactly 5 features, each referencing Q-ids, each with a Given/When/Then acceptance criterion,
  4 Non-goals, 5 Data model (Supabase SQL + RLS), 6 Screens and navigation, 7 Monetization (paywall placement, price
  copied from a named competitor if known, trial), 8 Definition of done, 9 Task plan (tasks ≤ 2h). Default stack line:
  Expo (React Native + TypeScript) + Supabase + RevenueCat + EAS. Save via `saveSpec`. `maxTokens: 8000`.
- `importItems(items: {body, url?, author?, appId?, source?, postedAt?}[])` — dedupe by hash, batch insert, return
  `{inserted, skipped}`; then callers enqueue `analyse_items`.
- `analyseItems(opts)` — like `analyseApp` but over `items` with `analysed_at is null or analysis_version < ANALYSIS_VERSION`;
  reuse `classifyBatch`/`normaliseItem` from `analysis.ts` (export what is needed). Then enqueue `group_labels`.

### `analysis.ts`
- After `analyseApp` finishes classifying, enqueue `group_labels` (dedup by enqueue).
- Export `classifyBatch` and `PendingReview` type for reuse.

### `apps.ts`
- `addApp`: if `store === 'android'` and another row exists with the same `store_id` (any country) → throw
  `"Google Play reviews are not per country. This app is already tracked; change its review language instead."`
- `setAppOwn(id, own: boolean)`.

### `jobs.ts`
- `JobType` += `"group_labels" | "generate_spec" | "analyse_items"`.
- `group_labels` and `analyse_items` have no appId; make them mutually exclusive with themselves: in `claimJob`, also skip a
  queued job whose `type` is one of those while a job of the same type is running (extend the `not exists` clause).

### `queries.ts`
- `navSummary()` add `opportunities` count (status not killed).
- `dashboardStats()` add `opportunities` (surfaced count) and `signals` (evidence with wtp_signal <> 'none', last 30 days).

### Tests (`packages/core/test`)
- `opportunities.test.ts` (unit): the model-reply validator (unknown ids dropped, forgotten labels become their own group,
  every label mapped once); score formula on a fixed fixture.
- `integration.test.ts`: extend the fake AI server to answer the grouping prompt (detect by `"labels"` key in the user JSON)
  and the spec prompt (detect by "SPEC" in system), then: `groupLabels` creates opportunities from the seeded reviews,
  `listOpportunities` returns them with `listings`/`score` > 0, `own='only'` is empty until `setAppOwn`, `importItems` dedupes,
  `analyseItems` classifies and the item shows in `getOpportunity().evidence`, `generateSpec` stores markdown containing
  "## 3" or "MVP", `setOpportunityStatus(..., 'killed')` hides it from the default list, `mergeOpportunities` repoints labels,
  android multi-country add is refused, `claimJob` never runs two `group_labels` at once.

## Worker (`apps/worker/src/index.ts`)
- cases: `group_labels` → `groupLabels`; `generate_spec` (payload `{opportunityId}`) → `generateSpec`; `analyse_items` → `analyseItems`.
- Nightly: after `sync_all` is queued by the cron, also enqueue `group_labels` (it is cheap when nothing is new).

## MCP (`apps/mcp/src/tools.ts`) — new tools (total becomes 23; update `apps/mcp/test/smoke.mjs` count and add calls;
update the `TOOLS` array in `apps/web/app/settings/page.tsx` and the README tool table)
- `list_opportunities` {status?, kind?, own?, query?, limit?} → stats rows (readOnly)
- `get_opportunity` {opportunity_id} → row + stats + evidence (limit param, default 50) + labels (readOnly)
- `search_reviews` {query?, signal?, label?, min_pain?, limit?, offset?} → cross-app evidence (readOnly)
- `save_gate_result` {opportunity_id, checks: record<string, boolean>, notes?} — the 8 gate keys:
  `scope, permissions, single_player, monetization, demand, distribution, data_legal, founder_fit`; unknown keys rejected
- `save_spec` {opportunity_id, spec_md}
- `set_opportunity_status` {opportunity_id, status, reason?, revisit_after?}
- `record_outcome` {opportunity_id, installs?, trial_starts?, paying?, notes?}
- `import_items` {items: [{body, url?, author?, app_id?, source?, posted_at?}]} → {inserted, skipped}; enqueues `analyse_items`
No tool calls the model; `generate_spec` is NOT an MCP tool (it is a worker job started from the web UI).

## Web (`apps/web`) — shadcn, compose existing `components/ui/*`, no ad-hoc restyling
- Sidebar (`components/app-sidebar.tsx`) + command menu: "Opportunities" (with count) and "Import".
- `/opportunities`: table/list sorted by score desc; filters: status (default: not killed), kind, "Competitors | My apps",
  text; columns: label, kind badge, score, evidence n, listings, avg pain, signal chips, last seen, status badge.
  A "Group new labels" button enqueues `group_labels` (server action) and `<JobWatcher>` polls.
- `/opportunities/[id]`: header (label editable inline, kind, status select, score/stat cards using `stat-card.tsx`),
  tabs: Evidence (list like reviews-panel with app icon + store + country, evidence_span highlighted, filters signal/min pain),
  Gate (8 checkboxes with the PASS/FAIL wording from the brief, notes, Save; shows pass count, warns that permissions /
  single_player / data_legal failures are permanent), Spec (Generate button → enqueues `generate_spec`, JobWatcher; renders
  markdown read-only in a `<pre>`/simple renderer; Copy and Download `.md`), Outcome (installs, trial starts, paying, notes,
  Save; kill criteria hint: <100 installs, <2% trial, <1 paying/100 installs at day 60). Kill dialog (reason, revisit after).
  Merge dialog (pick another opportunity from a select).
- `/import`: textarea (one item per blank-line-separated paragraph), optional source URL, source select, optional app
  select; submit → `importItems` + enqueue `analyse_items`; shows inserted/skipped.
- App page (`/apps/[id]`): "This is my app" switch (server action `setAppOwnAction`) in the details area.
- Dashboard (`/`): stat tiles for opportunities and signals (30d) linking to `/opportunities`.
- Server actions in `app/actions.ts`: `groupLabelsAction`, `generateSpecAction(id)`, `saveGateAction`, `saveSpecAction`,
  `setOpportunityStatusAction`, `recordOutcomeAction`, `updateOpportunityAction`, `mergeOpportunitiesAction`,
  `importItemsAction`, `setAppOwnAction`. All use the existing `run()` helper and `revalidatePath`.

## Docs
- README: new "Opportunities (H93)" section (English + a short Arabic paragraph in the Arabic guide): what an opportunity is,
  how grouping works, the score formula, the gate, spec generation, outcomes/graveyard, import, own apps, multi-country rule
  (iOS only), and the new MCP tools in the tool table. `CLAUDE.md`: one bullet on opportunities/evidence union and one on
  the multi-country rule.
