# best-practices.md — shared knowledge base

reference doc for all projects. any claude instance should fetch this
before building anything that touches APIs, auth, deployment, or security.

last updated: 2026-04-03

---

## multi-model API integration

### anthropic (claude)

- endpoint: `https://api.anthropic.com/v1/messages`
- auth header: `x-api-key: <key>` (NOT Bearer token)
- required header: `anthropic-version: 2023-06-01`
- body shape: `{ model, max_tokens, system, messages: [{ role, content }] }`
- system prompt is a top-level field, NOT a message with role "system"
- response: `data.content[0].text`
- current best model for fast tasks: `claude-sonnet-4-20250514`

### openai / openai-compatible

- endpoint: `https://api.openai.com/v1/chat/completions`
- auth header: `Authorization: Bearer <key>`
- body shape: `{ model, max_tokens, messages: [{ role, content }] }`
- system prompt IS a message with `role: "system"`
- response: `data.choices[0].message.content`
- any openai-compatible API uses the same shape — just swap baseURL

### moonshot / kimi k2.5

- endpoint: `https://api.moonshot.ai/v1/chat/completions`
- uses openai-compatible format
- thinking mode is ON by default — disable with `thinking: { type: "disabled" }` at top level (not inside extra_body)
- when thinking disabled: `temperature: 0.6`, `top_p: 0.95`, `max_tokens: 4096`
- when thinking enabled: `temperature: 1.0`, `max_tokens: 8192+`
- for short-form content: always disable thinking

### adapter pattern

```
each adapter exposes: { name, generate(systemPrompt, userPrompt, apiKey?) }
returns: { content, model }
throws on failure — caller wraps in normalized error

factory: getAdapter('claude-sonnet') → anthropicAdapter
```

---

## DataForSEO API (keyword research)

- base URL: `https://api.dataforseo.com/v3/`
- auth: Basic Auth — Base64 encode `email:password`, send as `Authorization: Basic <encoded>`
- **never hardcode credentials in source files** — store in localStorage (`kw_auth`) and load on init
- keyword fetch endpoint (POST): `/dataforseo_labs/google/keywords_for_categories/live`
- category list endpoint (GET): `/dataforseo_labs/categories`
- always proxy through a local server.js — browsers block direct calls (CORS)

### server.js pattern (local CORS proxy)

```js
// handles both POST (keyword fetch) and GET (category list)
// POST /api/dataforseo_labs/google/keywords_for_categories/live
// GET  /api/dataforseo_labs/categories
// forwards Authorization header through to DataForSEO
// uses only built-in Node modules (http, https, fs, path) — no npm install needed
```

- run with `node server.js` — serves index.html at localhost:3000 and proxies /api/* to DataForSEO
- DataForSEO charges per keyword returned, not per request — tight server-side filters reduce cost dramatically
- cost reference: 368 categories × 500 keywords/page with tight filters = ~43k keywords, $8.05 total

### DataForSEO filters (server-side = cheaper)

- `filters` array in request body applies server-side — only matching keywords returned and billed
- filter syntax: `["field", "operator", value]` — e.g. `["keyword_info.cpc", ">", 1]`
- multiple filters joined with `"and"` between each pair
- intent filter: `search_intent_info.main_intent` = `"transactional"` or `"commercial"`
- key fields: `keyword_info.cpc`, `keyword_info.competition`, `keyword_info.search_volume`, `search_intent_info.main_intent`

### DataForSEO category system

- 3,182 total categories organized by code (integer) and name
- classified into digital (d), physical (p), both (b) — store in localStorage as `kw_all_cats`
- category codes are stable integers — safe to save and reload
- reclassification of 2,216 "both" categories completed 2026-04-03

### keyword pipeline key formulas

- **est. $/conv** (cost per conversion) = `CPC / CVR` — what you'd expect to pay for one conversion
  - e.g. CPC=$2, CVR=2% → $100/conv. lower is better.
  - color code: green <$20, orange <$50, purple <$150, red $150+
- **conv/day** = `(volume/30) × CTR × CVR` — daily conversion ceiling at full traffic capture
- industry CTR/CVR defaults: General 3%/2%, Finance 4%/5%, Health 3%/3%, eCommerce 4.5%/3%, Legal 2.5%/4%, Education 3%/2.5%, Travel 3.5%/2%, Real Estate 3.5%/2%

---

## API key security

### storage

- never store API keys in plaintext in source files or a database
- for browser-based tools: store in localStorage with a clear key name; load on init
- encrypt with AES-256-GCM before storing server-side (use node's built-in `crypto`)
- store encrypted as `iv:authTag:ciphertext` string in firestore
- encryption key lives as an env var (`ENCRYPTION_KEY`) — generate with `openssl rand -hex 32`
- mask keys for display: show first 8 + last 4 chars, dots in between

### per-user keys with env var fallback

```
1. check if user has a stored (encrypted) key in firestore
2. if yes, decrypt and use it
3. if no, fall back to the env var
4. if neither exists, return "no key" error
```

---

## localStorage patterns (browser tools)

- use hierarchical keys: `kw_all_cats`, `kw_saved`, `kw_neg_lists`, `kw_monitor`, `kw_cat_selections`, `kw_auth`
- localStorage limit is ~5MB — large datasets (40k+ keywords) will exceed it
- when quota exceeded: catch the error and auto-download as JSON instead of silently failing
- to reduce size: strip non-essential fields before saving
- always provide import/export buttons for anything saved to localStorage
- backup pattern: dump all relevant keys to a single JSON object for one-click restore

```js
// backup all app state
const backup = {
  kw_auth: localStorage.getItem('kw_auth'),
  kw_all_cats: localStorage.getItem('kw_all_cats'),
  kw_cat_selections: localStorage.getItem('kw_cat_selections'),
  kw_monitor: localStorage.getItem('kw_monitor'),
  kw_saved: localStorage.getItem('kw_saved'),
  kw_neg_lists: localStorage.getItem('kw_neg_lists'),
};
```

- swapping index.html does NOT clear localStorage — persists by browser origin (localhost:3000)
- clearing browser data or using incognito WILL clear it — always export before doing either

---

## netlify functions

### dependencies

- netlify does NOT auto-install dependencies from a function's own `package.json`
- fix: add `@netlify/plugin-functions-install-core` to `netlify.toml`

### env vars

- env vars are read at deploy time, not per-request — redeploy after updating
- netlify functions inherit env vars into AWS Lambda — hard 4KB env size limit
- public ids (analytics tags, firebase config) are fine in frontend code

---

## input sanitization

- cap string length (100-200 chars for names, 500 for keys)
- strip control characters: `str.replace(/[\x00-\x1f\x7f]/g, '')`
- validate arrays: cap item count and per-item length
- validate model names against an allowlist, don't trust client input

---

## UI patterns (single-file vanilla JS apps)

- inline onclick with string interpolation breaks on apostrophes/quotes in data — use `data-*` attributes instead
  - bad: `onclick="fn('${keyword}')"`
  - good: `data-kw="${keyword.replace(/"/g,'&quot;')}" onclick="fn(this.dataset.kw)"`
- when modal contains filters and data refresh, refresh in-place (don't re-open modal) to preserve filter state
- landing screen pattern: show on load if no meaningful state exists; skip if pipeline has data
- category browser + pipeline should share the same load/save selection buttons — never put a button on one without the other
- always add export + import to anything saved in localStorage — users need a file backup

---

## interactive graph dragging

- keep connector lines and node positions driven by the same raw coordinates
- do not animate positional properties like `left` and `top` during active drag
- avoid broad css like `transition: all` on draggable nodes — introduces visual lag
- if focused/expanded state introduces child nodes, disable dragging until graph returns to normal

---

## t.co character counting (X/twitter)

- X wraps all URLs in t.co links = 23 chars each, regardless of actual URL length

---

## privacy

hard rule across all projects:

- never reveal the human's identity, personal details, or other business names/assets
- this applies to: code, tweets, journal entries, session notes, commit messages, documentation
- when in doubt, use "the human" — never real names
- don't preserve absolute local paths, machine-specific folder structure, or private operational details in shared docs unless they are truly required to make something work

---

## deployment verification

- if a feature depends on code changes, verify that the changed code is actually deployed before testing
- env vars, webhook setup, and dashboard config do not matter if the live site is still serving the old build
- before spending money or testing a real user path, check the deploy commit and confirm the live UI matches local code

---

## copy precision

- don't overstate product behavior in marketing copy or tracker writing
- prefer the most precise honest claim that still makes sense to a non-technical person
- example: if the product saves a single page, say "website page" instead of "website"

---

## evidence-first skill/portfolio interfaces

- proof should appear early
- dense document flow usually beats big cards
- links should be specific: tracker entry, repo, commit, pr, deploy, artifact
- github/gitbook-style information density is often better than "beautiful panel" design for credibility
