# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is a **single-file static web app**, not a project with a build system. The entire application (HTML, CSS, JS, data, Firebase SDK imports, service worker, PWA manifest) lives in one ~1100-line file. There is no `package.json`, no bundler, no tests, no linter, and no CI configuration.

- `index.html` — the app.
- `belissima-firebase.html` — a **byte-identical duplicate** of `index.html` (verified via `diff`). Any edit to the app must be mirrored to both files, or one of them should be deleted. Don't assume they serve different purposes.

## Running / developing

- There is no build step. Open `index.html` in a browser, or serve statically (e.g. `python3 -m http.server` from the repo root) and hit `http://localhost:8000`.
- A service worker is registered from a runtime-generated blob URL, and the PWA manifest is also generated at runtime as a blob (see end of the `<script>` block). When iterating locally, hard-reload or unregister the service worker to avoid stale cache behavior.
- The UI is hardcoded mobile-first at `max-width:430px` and assumes iOS-style chrome; test at phone widths.
- All UI text, comments, and category labels are in **Portuguese (pt-BR)** with BRL currency formatting via `Intl.NumberFormat('pt-BR', …)`. Preserve the language when editing user-facing strings.

## Domain model (read this before touching `calc()` or any mutation)

The app tracks a two-partner mining-company split (Belíssima Mineração). The partnership rule is the core invariant and is encoded in `calc(m)` (search for `// ─── CALC ───`):

```
vendas ÷ 3
  Eli   → 1/3  (liquid, untouchable — never debited)
  Paulo → 2/3  − despesas − reembolsos_eli − adiantamentos − repasses + saldoAnterior
```

Concretely, `calc()` returns `{tv, uma, duas, td, ePend, eReemb, ta, tr, sa, pLiq}` where `pLiq = duas − td − eReemb − ta − tr + sa`. Balance (`pLiq`) carries month to month via `DB.saldoAnterior[YYYY-MM]`.

Top-level `DB` collections (also the Firestore subcollection names under `users/{UID}/…`):

- `vendas` — sales
- `despesas` — expenses. Each has `quem: 'eli' | 'empresa'` (who fronted the cash) and `reemb: boolean` (whether Eli has been reimbursed). Eli-paid + not-yet-reimbursed expenses are what the "Acerto" flow settles.
- `adiantamentos` — advances already paid to Paulo
- `repasses` — profit transfers to Paulo
- `investimentos` — capex (imóveis/veículos/cripto/mineração)
- `saldoAnterior` — `{ 'YYYY-MM': number }`, prior-month carry

The **Acerto** flow (`openAcerto` / `togA` / `confirmarAcerto`) lets Eli select Eli-paid expenses to mark `reemb = true` in bulk — that's the only place `reemb` flips to true.

## Data persistence — two modes

The `USE_FIREBASE` flag switches between two storage paths. **Every mutation must go through both** when applicable:

1. **Demo/offline mode** (`USE_FIREBASE = false`, entered via "Entrar em modo demonstração"): state lives in `DB`, persisted to `localStorage` under key `belissima_db` via `lsLoad()` / `lsSave()`.
2. **Cloud mode** (`USE_FIREBASE = true`, entered via Google sign-in): each mutation calls `fbSave(col, item)` which writes `users/{UID}/{col}/{id}` in Firestore. Initial load is a one-shot `fbLoad()` on sign-in — there are **no `onSnapshot` listeners wired up** (despite being imported), so cross-device updates only appear on reload/re-login.

The canonical mutation pattern used throughout (see `saveL`, `saveRep`, `confirmarAcerto`) is:

```js
DB.<col>.push(item);           // or mutate in place
await fbSave('<col>', item);   // no-op if !USE_FIREBASE
lsSave();                      // always persist locally too
rAll();                        // re-render everything
```

Follow this order when adding new mutations. Skipping `lsSave()` or `rAll()` silently breaks demo users / leaves the UI stale.

## Firebase configuration

The `firebaseConfig` object at the top of the module script (around line 33) ships with `"COLE_AQUI"` placeholders. The sentinel `FIREBASE_CONFIGURED = firebaseConfig.apiKey !== "COLE_AQUI"` gates whether Firebase initializes at all; when not configured, `window._fbOK = false` and the Google sign-in button shows a config warning. Do not commit real Firebase keys to this repo — they're meant to be pasted in locally before deployment.

## Rendering model

There's no framework. Views are five `.pane` divs (`#p-home`, `#p-vendas`, `#p-despesas`, `#p-paulo`, `#p-invest`) toggled by `navTo(tab)`, and each has a dedicated render function (`rHome`, `rVendas`, `rDespesas`, `rPaulo`, `rInvest`) that rebuilds its DOM via `innerHTML` from the in-memory `DB`. `rAll()` re-renders the current pane plus home. After any data change, call `rAll()`.

The period filter (`PER` ∈ `'dia' | 'semana' | 'mes' | 'tudo'`) and selected month (`MES` = `'YYYY-MM'`) are module-level globals; renders filter via `inP(item)`.

## AI category suggestion

`aiCat(txt)` in the `// ─── LANÇAR ───` section is **not an LLM call** — it's a debounced keyword matcher against the `AI_RULES` table that auto-selects a category and `debita de quem` based on the description. The spinner + "✦ IA" badge are cosmetic; add new categorizations by appending to `AI_RULES`, not by calling an API.

## Conventions worth preserving

- Short, terse identifiers (`R`, `fD`, `pV`, `rAll`, `txH`, `kc`, `vc`, `ti`, etc.) are used pervasively in both JS and CSS class names. Don't rename them for "clarity" — they're referenced by string throughout the single file and the tradeoff is intentional for file size.
- CSS custom properties at `:root` (search `--or1`, `--dk`, `--gr`) define the entire palette; use them instead of hardcoded colors.
- IDs are globally namespaced by a short prefix per pane (`h-*` home, `v-*` vendas, `d-*` despesas, `p-*` paulo, `i-*` invest, `kh-*`/`kd-*` KPI cards, `l-*` lançar sheet, `ac-*` acerto, `rep-*` repasse). Match the prefix when adding new DOM.
- Dates are stored as `'YYYY-MM-DD'` strings and compared lexically. Month filtering uses `data.startsWith('YYYY-MM')`. Don't convert to `Date` objects for filtering.
- Currency input uses `<input type="number">` with `inputmode="decimal"`; `pV(v)` is the safe parser.

## Git / branching

The branch for documentation work in this task is `claude/add-claude-documentation-x5enT`. The repo has only two commits total (`init`, `Add files via upload`) — there is no established commit-message convention to follow.
