# H93 Market (Appllama) — build spec

Repository `/home/user/storefront-`, branch `h93/market-appllama` (from `main`). Same conventions and checks as the earlier
specs (`CLAUDE.md`: postgres.js tagged SQL, batched writes, migrations DDL-only and numbered, long work as worker jobs,
client components import only types from `@lens/core`). Local Postgres 16 + pgvector on 127.0.0.1:5433.

Checks before a stage is done:
```
cd /home/user/storefront- && pnpm format:check && pnpm typecheck && TEST_DATABASE_URL=postgres://postgres@127.0.0.1:5433/lens_test pnpm --filter @lens/core test && pnpm build
```

## What this adds

A **Market** source: the user's Appllama Pro subscription (hosted MCP server, OAuth) is connected once from Settings; then a
`/market` page searches Appllama's catalogue, shows which apps are already saved / in the library, and saves chosen apps:
profile (revenue, downloads, IAP prices, rank, onboarding flows…) and **every screen** (image stored locally — Appllama media
URLs expire in ~1 h). Saving an app that is not in the library adds it (store `ios`, `store_id` = Appllama `app_id`, which
IS the iOS track id — verified: Calm = 571800810) so reviews and market data live on one app record. Market data then feeds
opportunities (real competitor prices, revenue), the SPEC's monetization section and the gate's demand check.

Appllama facts (verified 2026-10-08):
- MCP endpoint `https://mcp.appllama.io/mcp`, Streamable HTTP. OAuth metadata at
  `https://mcp.appllama.io/.well-known/oauth-authorization-server`:
  `authorization_endpoint /authorize`, `token_endpoint /token`, `registration_endpoint /register` (dynamic client
  registration), `revocation_endpoint /revoke`, scope `appllama`, `response_types [code]`, grants
  `authorization_code, refresh_token`, token auth `client_secret_post | client_secret_basic`, PKCE `S256`.
- Every tool call = 1 credit; `get_credits` free. Limits: 90/min, 400/day, 1500/month (Pro). Terms forbid harvesting the
  catalogue; saving specific study targets is the intended use. Media carries a watermark (reference only).
- Tool payloads (exact shapes):
  - `search_apps {query?, sort?, cursor?, launched_after?, launched_before?, downloads_min?, downloads_max?, revenue_min?,
    revenue_max?, rating_min?, rating_max?, price_min?, price_max?, onboarding_steps_min?, onboarding_steps_max?, board_id?}`
    → `{ apps: AppSummary[], total, next_cursor, credits: {spent, remaining_this_month} }`
  - `AppSummary = { app_id, name, subtitle, publisher, categories[], rating:{average,count}, category_rank:{rank,category},
    revenue:{display, monthly_usd, as_of} | nulls, downloads:{display,value}, in_app_purchases:[{title,duration,price}],
    launched, last_updated, screens_count, videos_count, flows:[{name,screens}] }`
  - `get_app {app_id}` → AppSummary + `description, current_version, publisher_country, ratings_breakdown:{"1".."5"},
    top_countries[], languages[], sections:[{section,label}], category_rank.country/as_of`
  - `list_app_screens {app_id, cursor?, flow?, section?}` → `{ app:{app_id,name}, screens: Screen[], total, next_cursor }`,
    10 per page. `Screen = { screen_id, name, flow, section (welcome-screen|onboarding|paywall|other-tabs), position,
    kind (image|video), media_url (expires ~60 min), width, height, duration_ms, dominant_color, colors[], ui_elements[] }`
  - `get_credits {}` → `{ period_start, resets_on, monthly_credits, bonus_credits, used, remaining, limits:{per_minute,per_day} }`
  - `list_my_boards {}` → `{ boards: [{board_id, name, kind: screens|apps|flows, item_count…}] }`;
    `get_board {board_id, cursor?}` → apps boards return AppSummary-shaped items.
- MCP tool results arrive as `content: [{type:"text", text:"<json>"}]`; parse the text.

## Migration `db/migrations/0007_market.sql` (DDL only)

```sql
create table if not exists app_market (
  app_id uuid primary key references apps(id) on delete cascade,
  appllama_id text not null unique,
  profile jsonb not null,                 -- the get_app payload (minus hint/credits)
  revenue_monthly_usd numeric,
  downloads numeric,
  rating numeric, ratings_count bigint,
  category_rank int, category text,
  launched date, last_updated date,
  screens_count int, videos_count int,
  screens_synced int not null default 0,  -- how many screens are stored locally
  screens_cursor text,                    -- resume point when a save was interrupted
  fetched_at timestamptz not null default now(),
  screens_fetched_at timestamptz,
  credits_spent int not null default 0
);
create table if not exists app_screens (
  id bigserial primary key,
  app_id uuid not null references apps(id) on delete cascade,
  screen_id text not null,
  name text, flow text, section text, position int,
  kind text not null default 'image',
  path text,                              -- stored WebP relative to MEDIA_DIR (null for videos / failed downloads)
  width int, height int, duration_ms int,
  dominant_color text, colors text[] not null default '{}', ui_elements text[] not null default '{}',
  fetched_at timestamptz not null default now(),
  unique (app_id, screen_id)
);
create index if not exists app_screens_app on app_screens (app_id, section, position);
alter table app_market enable row level security;
alter table app_screens enable row level security;
```

## Settings

```ts
appllama: {
  clientId: string; clientSecret: string;              // from dynamic registration
  accessToken: string; refreshToken: string; expiresAt: string | null; scope: string;
  connectedAt: string | null;
  usage: { day: string; calls: number };               // local daily counter (YYYY-MM-DD UTC)
  mcpUrl: string;                                      // default "https://mcp.appllama.io/mcp"; tests override
}
```
`saveSettings("appllama", patch)` merges. Never expose tokens/secret to client components: the Settings page shows only
"Connected since …" and the credit balance.

## Core — `packages/core/src/appllama.ts` (Stage M1)

Dependencies: add `@modelcontextprotocol/sdk` (same version as apps/mcp: 1.30.1) to packages/core.

### OAuth (all exported)
- `appllamaMetadata(mcpUrl)` → fetch `${origin}/.well-known/oauth-authorization-server` (cache in module for 10 min).
- `beginAppllamaConnect(redirectUri)` → ensures a client (if `clientId` empty: POST `registration_endpoint` with
  `{ client_name: "Storefront Lens", redirect_uris: [redirectUri], grant_types: ["authorization_code","refresh_token"],
  response_types: ["code"], token_endpoint_auth_method: "client_secret_post", scope: "appllama" }` → save `client_id`,
  `client_secret`), generates PKCE verifier/challenge (S256) + `state`, stores `{verifier, state, redirectUri}` in settings key
  `appllama_pending` (short-lived), returns the authorize URL
  (`response_type=code&client_id&redirect_uri&scope=appllama&state&code_challenge&code_challenge_method=S256`).
- `finishAppllamaConnect({code, state})` → validates state, POSTs `token_endpoint` (`grant_type=authorization_code`,
  `code`, `redirect_uri`, `client_id`, `client_secret`, `code_verifier`), stores tokens + `expiresAt` (now + expires_in) +
  `connectedAt`, clears `appllama_pending`.
- `refreshAppllamaToken()` → `grant_type=refresh_token`; called automatically when `expiresAt` is within 60 s or a call
  returns 401 (one retry).
- `disconnectAppllama()` → best-effort POST `revocation_endpoint` for the refresh token, then clears tokens
  (keeps clientId/secret).
- `appllamaConnected(settings)`.

### MCP client
- `appllamaCall(tool, args)` → opens an MCP `Client` with `StreamableHTTPClientTransport(new URL(mcpUrl), { requestInit:
  { headers: { authorization: "Bearer <accessToken>" } } })`, calls the tool, parses `content[0].text` as JSON, closes.
  Before each call: pace to ≤ 80/min (sleep when needed; skip pacing when `mcpUrl` is not the production host — tests),
  bump `appllama.usage` (reset when the UTC day changes) and refuse with a clear error when `usage.calls >= 390`
  ("Appllama daily limit nearly reached (390/400). Try again tomorrow."). Convert 401 → refresh once → retry; 402/429 →
  friendly error. Keep one client per call (stateless; simplest and fine at this volume).
- `appllamaCredits()` (free), `appllamaSearch(params)` (returns the payload plus, per app, `saved: boolean` and
  `library_app_id: string | null` by joining on `app_market.appllama_id` / `apps.store='ios' and store_id=app_id`),
  `appllamaBoards()`, `appllamaBoard(boardId, cursor?)`.

### Saving (the job logic)
- `saveMarketApp(appllamaId, opts: { screens?: boolean (default true), log?, onProgress?, signal? })`:
  1. `get_app` → upsert `apps` (`store='ios'`, `store_id=appllamaId`, `country='us'`, name/developer/category/description
     filled only when the row is new or empty — never overwrite store-synced fields) via `addApp` when missing (which also
     queues the first store sync), then upsert `app_market` (profile + the denormalised columns).
  2. If `screens`: walk `list_app_screens` from `screens_cursor` (resume) page by page; for each screen of kind `image`
     download `media_url` immediately with `downloadImage` and `storeImage(bytes, `${appId}/market`)`; videos: store
     metadata, `path` null (do not download). Upsert `app_screens` per page in one statement; update `screens_cursor` /
     `screens_synced` / `credits_spent` after each page so an interrupted job resumes without re-paying. Clear the cursor and
     set `screens_fetched_at` at the end. Honour `signal`.
  3. Return `{ appId, screens: n, creditsSpent }`.
- `estimateCredits(app: {screens_count})` = `1 + ceil(screens_count / 10)` (exported, pure).
- `refreshMarketApp(appId)` = `saveMarketApp` for an already saved app (re-walks screens: resets cursor).
- Queries: `getMarket(appId)` → `app_market` row + `screens` grouped in journey order (`section` order welcome-screen,
  onboarding, paywall, other-tabs; then position); `marketSummaryFor(appIds)` for the opportunity page (name, revenue,
  downloads, rating, cheapest monthly/annual IAP price parsed from `in_app_purchases`); `listSavedMarket()` for the
  Market page's "Saved" column and the `saved` flags.
- `iapPrices(profile)` (pure, exported): from `in_app_purchases` derive `{ monthly: min price with duration Monthly,
  annual: min Annual, weekly: min Weekly, lifetime: min Unknown with "lifetime" in title }` as numbers (USD) or null.

### Opportunities / spec hooks (small edits in `opportunities.ts`)
- `getOpportunity` adds `market: MarketSummary[]` for the distinct apps in its evidence (via `marketSummaryFor`).
- `generateSpec`: when any evidence app has market data, add a "Market data" block to the prompt (name, revenue/month,
  downloads, monthly/annual price) and tell the model to copy the price from the closest competitor. `generateValidation`
  same.
- Gate helper `demandHint(opportunity)` (pure): returns `"pass"` when some evidence app has `ratings_count ≥ 1000 && rating ≤
  3.8` OR the opportunity has `n ≥ 15` evidence across `listings ≥ 2`; else `"unknown"`. Exposed on `getOpportunity` as
  `gate_hints: { demand }`.

### Tests
- `test/fake-appllama.ts`: an in-process MCP server (`@modelcontextprotocol/sdk` server + Streamable HTTP, node `http`)
  that requires `Authorization: Bearer good-token` (401 otherwise), implements `search_apps`, `get_app`, `list_app_screens`
  (2 pages for one app, image screens served by the same server at `/media/<id>.png` generated with sharp), `get_credits`,
  `list_my_boards`; plus OAuth endpoints `/.well-known/oauth-authorization-server`, `/register`, `/authorize` (not used),
  `/token` (authorization_code with PKCE check → `good-token` + refresh; refresh_token → `good-token-2`), `/revoke`.
- `appllama.test.ts` (integration, needs DB): registration + token exchange store settings; expired token refreshes
  automatically; `appllamaSearch` marks saved apps; `saveMarketApp` creates the library app, stores profile + all screens
  with WebP files on disk, resumes from a cursor after an aborted run without duplicating screens, `estimateCredits`,
  `iapPrices`, daily counter refusal at 390, `getMarket` ordering, `disconnectAppllama` clears tokens.

## Stage M2 — worker, MCP, web
- Jobs: `market_save` {appllamaId, screens} → `saveMarketApp`; `market_refresh` {appId} → `refreshMarketApp`. Both carry
  `appId` in the payload when known so claimJob serialises per app. Add to `JobType`; `activeJobs` already exposes app_id.
- MCP tools (3 → 32 total): `market_search` (same params as search_apps + returns `saved`/`library_app_id`; readOnly),
  `market_save` {appllama_id, screens?} (enqueues `market_save`, returns the credit estimate), `get_market` {app_id}
  (profile + screens with absolute media URLs; readOnly). Update smoke test count (32) and add calls against the fake
  server? The smoke test has no Appllama; assert `get_market` on an unsaved app returns a tool error and `market_search`
  returns a clear "not connected" error.
- Web:
  - `app/api/appllama/connect/route.ts` → `beginAppllamaConnect(`${PUBLIC_URL}/api/appllama/callback`)` → redirect;
    `app/api/appllama/callback/route.ts` → `finishAppllamaConnect` → redirect to `/settings#appllama` with `?connected=1`
    or `?error=…`.
  - Settings card **Appllama**: status (Connected since… / Not connected), credits (remaining this month, used today from the
    local counter), Connect / Disconnect buttons, a note that saving an app costs 1 + screens/10 credits, and the
    non-harvesting note.
  - `/market` page: search box + filters (sort, revenue min, rating min, launched after, price max, onboarding steps max),
    results as cards (icon not available — use the first letter avatar like the app list; name, publisher, revenue display,
    downloads display, rating ★ + count, cheapest monthly/annual price, screens count, flows chips (first 4), launched).
    Badges: **Saved** (link to the app's Market tab), **In library** (link to app). Button "Save (≈N credits)" → enqueues
    `market_save`; a confirm dialog when N > 15. Pagination via cursor ("Load more"). A top bar with credits (from
    `appllamaCredits`, cached 60 s in a settings key to keep it cheap… it is free, so just call it) and "Import a board"
    (select from `list_my_boards`, apps boards only → bulk `market_save` with the total estimate; max 20 per batch).
    `<JobWatcher>` so Saved badges update when jobs finish. Not-connected empty state with a Connect button.
  - App page: new tabs **Market** (stat cards revenue / downloads / rating / rank / launched; IAP table deduped by
    title+duration+price; top countries; languages; onboarding steps = screens in `onboarding` section; Refresh button
    (enqueues `market_refresh`, shows credit estimate)) and **Screens** (gallery grouped by section → flow, each tile the
    stored image via `mediaSrc`, name, click → lightbox like screenshots with UI elements + colors listed; videos show a
    placeholder with duration). Hide both tabs when no market row; show a "Find on Appllama" link to `/market?q=<name>`.
  - Opportunity page header: "Competitors in market" strip — for evidence apps with market data: name, revenue/mo,
    cheapest monthly price (and link). Gate tab: show the `demand` hint next to the Demand switch ("auto-check: pass —
    Calm has 1.9M ratings…" or "unknown").
  - Sidebar + command menu: "Market".
- Docs: README (Arabic + English): connecting Appllama (OAuth, no API key), what is saved, credits, limits, harvesting note,
  the three MCP tools; CLAUDE.md bullets (appllama.ts owns OAuth + MCP client; never log tokens; media URLs expire).

## Stage M3 — verification
Browser pass on `/market` (not-connected state), Settings card, app Market/Screens tabs with data seeded through the fake
server in a dev run, opportunity strip; full checks; PR.
