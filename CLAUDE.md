# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is a **single-page app shipped as a single `index.html`** (~6,800 lines) plus one Vercel serverless function (`api/ai.js`). There is no build step, no package.json, no test suite, and no linter. Deployment is static hosting on Vercel (production URL `sunny-sistema.vercel.app`). To "run" locally you just open `index.html` in a browser — but it depends on Firebase (live Firestore project `sunny-cleaning`) and on `/api/ai` for AI features, so against-prod testing is the normal workflow.

Almost every edit happens inside `index.html`. Treat it as the application source even though it is also the served artifact.

## How `index.html` is structured

The runtime is React 18 via UMD CDN bundles plus `babel-standalone`. The app's JSX source lives inside `<script type="text/plain" id="app-jsx">…</script>`; the final `<script>` in the file calls `Babel.transform(...)` and `new Function(code)()` to execute it at page load. Consequences:

- There is no module system — everything is top-level in one closure. Helpers, components, and the `App` body all share scope.
- Syntax errors render as a red `<pre>` over the root; check the browser console.
- React, ReactDOM, Firebase (`firebase-app-compat`, `firebase-firestore-compat`, `firebase-auth-compat`), QRCode, and Babel are loaded from CDNs in `<head>`. The Firestore + Auth handles are exposed as the globals `db` and `auth`.
- A service worker registered inline at the bottom of `index.html` caches the shell under the name `sunny-v4` (was `sunny-v3` before the 2026-05-19 cleanup). **Bump that cache name when shipping changes that must invalidate clients**, otherwise returning users will see the old HTML.

### Entry / routing

`AppWrapper` (around line 400) chooses one of four entry points based on URL params parsed at the very top of the file:

| URL param | Global | Page rendered | Auth |
|---|---|---|---|
| `?f=<token>` | `window.__FB_TOKEN__` | `FeedbackPublicPage` | none (public survey) |
| `?cad=<token>` | `window.__CAD_TOKEN__` | `EmpRegisterPublicPage` | none (public employee onboarding) |
| `?s=<code>` / `?scan=` | `window.__SCAN__` | normal `App`, opens scan modal | normal auth |
| (none) | — | `LoginScreen` → `App` | Firebase Auth |

The role/permission lookup after sign-in reads the `users` Firestore key and supports plain role strings, `"cliente:<Client Name>"` for restricted client logins, or `{ role, tabs, client }` for custom-tab users.

Inside `App`, navigation is a single `vw` state variable. Each value (`dashboard`, `payroll`, `invoice`, `history`, `finance`, `report`, `tax`, `inventory`, `team`, `inspect`, `laundry`, `agenda`, `deepclean`, `manual`, `escala`, `feedback`, `strategy`, `database`, `icalsync`, `preview`, `payslip`) maps to a section in a long JSX expression gated by `vw==="…"`. The bottom-nav buttons (near the end of the file) call `setVw(...)`. When adding a new tab, both the renderer block and the bottom-nav array need entries; the `users.tabs` permission system also gates visibility for custom-role users.

## Data layer

There is no backend other than Firebase Firestore. All persistent state lives in **a single collection named `sunny`**, where each "table" is one document keyed by a short string (`db3`, `em3`, `sp3`, `si3`, `fn3`, `ag3`, `ep3`, `insp3`, `ld_h2`, `ld_lots`, `fb_campaigns`, `users`, etc.). Each doc stores `{ value: <JSON string>, updated: <ISO date> }`.

All reads/writes go through the `FB` helper near the top of the JSX block:

```js
FB.get(key)         // -> { key, value } | null
FB.set(key, value)  // value is auto-stringified if not a string
FB.del(key)
```

Inside `App`, the `sv(key, data)` wrapper is what you actually call from handlers — it JSON-stringifies, sets the save-flash indicator, and **refuses to overwrite a `CRITICAL_KEYS` entry with an empty array** (see `CRITICAL_KEYS = ["db3","em3","sp3","si3","fn3","ag3","ep3","insp3"]`). There is also `dsv(key, data, ms)` which debounces writes and mirrors to `localStorage._pend_<key>` so a tab close before flush is recovered on next load.

When changing data shapes, remember:

- Migration runs in the giant initial `useEffect` (around line 719) where each key is loaded one by one — that is the right place to add legacy-data fixups (see e.g. `ld_h2` merge logic, auto-generated `cd` codes on `db3`, lot status reconciliation).
- The seeded constants `D` (properties), `IE` (employees), `DC` (company info), `IPROD` / `ILINEN` (inventory) provide initial values when a key is missing in Firestore. Don't repurpose them as the source of truth — production data has long since diverged.
- Keys ending in `2` / `3` are versioned migrations; introducing a new schema for an existing module conventionally means a new suffix and a fallback read of the old one.

## Error handling — `svT` / `safe` / `popup` (in-progress migration)

> **Próxima sessão deve continuar pelo COMMIT 5.2 conforme `ERROR-HANDLING-AUDIT.md`.** A migração está pausada — ver `SESSION-SUMMARY-2026-05-19-paused.md` para o estado exato e a lista de handlers pendentes.

An incremental migration is underway to give the UI real feedback when persistence fails (previously `sv()` and `FB.*` silently swallowed errors and handlers showed optimistic success toasts). Four opt-in helpers live near `fl()` (around line 773):

| Helper | What it does |
|---|---|
| `friendlyErr(e)` | Translates an `Error` into a friendly pt-BR message. Branches: offline / network failure → "Sem conexão com a internet…"; CRITICAL_KEYS empty-array protection → "Operação cancelada por segurança…"; `AbortError`/timeout → "Demorou demais para responder…"; `QuotaExceededError` → "Memória do navegador cheia…"; permission-denied → "Sem permissão para essa ação…"; fallback → "Tente de novo em alguns segundos…". |
| `svT(k, d)` | Same shape as `sv` but **throws** instead of swallowing. Detects `FB.set` returning `null` (network fail) and the empty-array guard for `CRITICAL_KEYS`. |
| `safe(fn, okMsg, errLabel)` | `try { await fn(); if (okMsg) fl(okMsg); return true; } catch (e) { fl("❌ " + errLabel + " — " + friendlyErr(e)); return false; }`. Caller branches on the return: `if (!ok) return;` to skip form-clear / navigation. |
| `popup(label)` | Replacement for `var w = window.open("", "_blank")`. Returns `null` and shows a toast asking the user to allow popups when blocked. |

**Do not change `sv()` or `FB.get/set/del`.** They stay silent on purpose to support the ~50 callers that haven't been migrated yet (boot loads tolerate missing keys; auto-backup is best-effort; etc.). Migration is **opt-in per handler** — change `await sv(...)` to `await svT(...)` inside a `safe(...)` block, with `if (!ok) return` before any state cleanup or navigation. See `ERROR-HANDLING-AUDIT.md` and `SESSION-SUMMARY-2026-05-19-erro-handling.md` for the migration backlog.

### Handlers already migrated

- `savP` (linha ~799, save payroll) — proof of concept, commit `a43f0e2`.

### Handlers still using the silent `sv` (do not break — migrate explicitly)

- All other save handlers in the file: `savI`, `emitirInv`, `pagarInv`, `delP`, `delI`, `addFin`, `delFin`, `updateFinEntry`, `dupFin`, `syncFin`, `updatePStatus`, `uPr`/`uLn`/`uLB`/`uLoss`/`uCC`/`uCU`, `uInsp`/`uAg`/`uTk`, `setAgendaCleanStatus`, `autoGenPayrolls`/`autoGenInvoices` (fire-and-forget), iCal sync handlers, auto-faturamento triggers in boot. These should be migrated handler by handler with a browser test between commits.
- 22 PDF generators using `var w = window.open("", "_blank"); if (!w) return;` still silently fail when popup-blocked. Replace with `popup(label)` in a later commit.

### Boot useEffect

Intentionally **stays** on the silent pattern (`try { ... } catch {} `). Failure to load a Firestore key falls back to seed defaults — that's the right UX. Don't migrate the ~30 boot reads to throw.

## AI integration

`api/ai.js` is a thin Vercel serverless proxy to the Anthropic Messages API. It expects `process.env.ANTHROPIC_API_KEY` (set in Vercel project settings, not in code) and is hardcoded to model `claude-sonnet-4-5` with a 60s `maxDuration`. The frontend calls it as `POST /api/ai` with `{ system, messages, max_tokens }` — used by the Strategy view, SOP chat, and feedback insights.

Do not put the API key in `index.html`. Any AI call must go through `/api/ai`.

## iCal sync

The `icalsync` view fetches per-house iCal URLs (Airbnb/Vrbo/Booking) and creates Turnover events on the agenda. Because those hosts don't send CORS headers, the fetch path first tries a direct request and falls back to `https://api.allorigins.win/raw?url=…` as a public CORS proxy (`fetchICalURL`, around line 972). The sync is manual only — there is no cron; the user clicks **Sincronizar Agora**.

Turnovers created by iCal sync use the local property's `cd` code/client/price; only dates come from the remote calendar.

## Conventions worth knowing before editing

- **No tooling.** No prettier, no eslint, no TypeScript, no tests. The codebase intentionally keeps very tight one-liner style with single-letter variable names (`db`, `em`, `co`, `sP`, `sI`, `pI`, `iI`, `T`, `vw`, …) and inline-styled JSX. Match the surrounding density when adding code in a section, don't refactor toward a different style.
- **Theming.** `T` is the active palette (`TD` dark / `TL` light). Every styled element pulls colors from `T.bg / cd / ac / tx / dm / bd / rd / gr / or / pu / bl`. Don't hardcode hex unless you check both modes.
- **Language.** UI text is Portuguese (pt-BR) by default. The public feedback flow is the only place with full pt/en/es i18n (`FB_UI_TEXT` map). Toasts use `fl("…")`.
- **Money / dates.** `fm(n)` formats USD-style numbers; `fmD(yyyy-mm-dd)` formats display dates as mm/dd/yyyy; `getLocalDate()` / `getLocalDateOffset(d)` produce `YYYY-MM-DD` strings in local time (avoid `new Date().toISOString().slice(0,10)` — it shifts to UTC).
- **Destructive actions.** Use `safeDel(msg, onConfirm, type, item)` so the deleted record lands in the in-app trash (`_trash` key) and can be restored.

## Deployment

Push to the `origin/main` branch on `github.com/aluxester-star/sunny-sistema` — Vercel auto-deploys. There is no staging environment. Required Vercel env var: `ANTHROPIC_API_KEY`. The Firebase config in `index.html` is the public web config and is fine to commit.
