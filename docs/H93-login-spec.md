# H93 login — build spec

Repository `/home/user/storefront-`, branch `h93/login` (from `main`). Same conventions/checks as before. Replaces the
HTTP Basic Auth popup with a real single-user login: first-run password setup, a login page, signed session cookies,
optional per-device PIN unlock, rate limiting, logout, and a Security card in Settings. Cloudflare Access stays optional
as an outer layer. The MCP server keeps its bearer token (unchanged).

Checks: `pnpm format:check && pnpm typecheck && TEST_DATABASE_URL=postgres://postgres@127.0.0.1:5433/lens_test pnpm --filter @lens/core test && pnpm build`

## Model
- Settings key `auth` (never sent to client components):
  ```ts
  auth: {
    passwordHash: string            // "scrypt$<saltB64>$<hashB64>" (Node crypto.scrypt, N=16384, r=8, p=1, 64 bytes)
    passwordSetAt: string | null    // ISO; bumping it invalidates older sessions (epoch)
    devices: { id: string; name: string; pinHash: string; createdAt: string; lastUsedAt: string | null; failed: number }[]
    failed: { count: number; until: string | null }   // password attempts (single user ⇒ global counter)
  }
  ```
- `SESSION_SECRET` env (required for the web app; ≥32 chars). `.env.example` gains it with the `openssl rand -hex 32`
  hint; docker-compose already passes `.env` through — verify and add to the `web` service if needed.
- Session cookie `lens_session`: `base64url(json) + "." + base64url(HMAC-SHA256(secret, json))` where
  `json = { iat, exp, epoch: passwordSetAt, dev?: deviceId }`. httpOnly, `SameSite=Lax`, `Secure` when `PUBLIC_URL` is https,
  path `/`. Lifetime: 30 days with "Remember me", else 12 hours.
- Device cookie `lens_device`: random 32-byte id (base64url), 180 days, httpOnly, Lax, Secure as above. Only set when the user
  enables a PIN on that device.

## Core — `packages/core/src/auth.ts` (no new dependencies)
- `hashPassword(pw)`, `verifyPassword(pw, hash)` (timing-safe), `hashPin(pin)` (same scrypt), `verifyPin`.
- `passwordConfigured()`, `setPassword(newPw, { current?: string })` (requires current when one exists; min 8 chars;
  bumps `passwordSetAt`; clears `failed`), `checkPassword(pw)` → `{ ok, lockedUntil? }` applying the lockout:
  5 failures → 1 min, then doubling up to 15 min; success resets.
- Devices: `addDevice({name, pin})` → `{ id }` (PIN exactly 6 digits), `listDevices()` (no hashes), `removeDevice(id)`,
  `checkPin(deviceId, pin)` → `{ ok, removed? }` (5 wrong PINs removes the device), `touchDevice(id)`.
- `authEpoch()` → `passwordSetAt`.
- Tests `test/auth.test.ts` (DB): setup → verify → wrong password lockout escalation → change password bumps epoch →
  device add/check/remove and auto-removal after 5 failures → PIN validation.

## Web
- `apps/web/lib/session.ts` — dependency-free, **Web Crypto only** (it runs in `proxy.ts`): `signSession(payload, secret)`,
  `verifySession(cookieValue, secret)` → payload or null (bad signature / expired), `cookieOptions()`. Unit-testable but
  no test harness in web; keep it tiny.
- `apps/web/proxy.ts` (replaces Basic Auth): allow `/login`, `/setup`, `/api/health`, `/api/auth/*`, `/_next/*`,
  `/favicon.ico`; everything else requires a valid `lens_session` (signature + exp) else redirect to
  `/login?next=<path>` (API/media requests get 401 JSON instead of a redirect). No DB access here.
- `app/layout.tsx` (root): after proxy, call `requireSession()` (server helper in `apps/web/lib/auth-server.ts`) which
  reads the cookie, verifies it and compares `epoch` to `authEpoch()`; mismatch → `redirect("/login")`. `/login` and
  `/setup` use a separate minimal layout (`app/(auth)/layout.tsx`) without the sidebar.
- `/setup`: shown only while `!passwordConfigured()` (otherwise redirect `/login`). Form: password + confirm → sets the
  password, signs in, redirects `/`. The login page redirects to `/setup` when no password exists.
- `/login`: password form (+ "Remember me"); when a `lens_device` cookie matches a device with a PIN, show the **PIN form
  first** (6 boxes / numeric input, autofocus) with "Use password instead". Errors: wrong password, locked until …,
  PIN removed after 5 failures. On success: set `lens_session` (and `touchDevice`), redirect to `next` (same-origin paths
  only). Server actions `loginAction`, `pinLoginAction`, `logoutAction`.
- Logout: sidebar footer item + Settings; clears the session cookie (keeps the device cookie).
- Settings → **Security** card: change password (current/new/confirm); "Enable PIN on this device" (name auto from UA,
  6-digit PIN twice) → sets device cookie; devices table (name, created, last used, Remove); note that PINs are per
  device and a lost device should be removed here; Logout.
- `/api/appllama/callback` is behind the session like everything else (Lax cookie survives the OAuth redirect).
- Remove `BASIC_AUTH_*` from proxy, `.env.example`, README; diagnostics unaffected.
- README (Arabic + English): the new login, first-run `/setup`, `SESSION_SECRET`, PIN per device, lockout, Cloudflare
  Access still recommended. CLAUDE.md: auth bullets (proxy = signature only, layout = epoch check; `auth` settings never
  reach the client; session helpers are Web Crypto so proxy can use them).

## Verification (M3, done by the supervisor)
Browser: fresh DB → any page redirects to `/setup` → set password → logged in → logout → `/login` → wrong password ×5
→ locked message → correct password after lockout → Settings enable PIN → logout → `/login` shows PIN form → wrong PIN
→ correct PIN → in; change password → old session invalid on next request; `/media/...` without cookie → 401;
`/api/health` open.
