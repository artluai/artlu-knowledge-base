# best-practices.md — shared knowledge base

reference doc for all projects. any claude instance should fetch this
before building anything that touches APIs, auth, deployment, or security.

last updated: 2026-04-06 (external writes confirmation rule added)

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
- any openai-compatible API (moonshot, together, etc.) uses the same shape — just swap `baseURL`

### moonshot / kimi k2.5

- endpoint: `https://api.moonshot.ai/v1/chat/completions`
- uses openai-compatible format
- **thinking mode is ON by default** — kimi will "think" internally before responding, consuming tokens and time (30+ seconds)
- to disable thinking: add `thinking: { type: "disabled" }` to request body
  - this goes at the top level of the JSON, not inside `extra_body`
  - `extra_body` is a python/node SDK wrapper — raw fetch calls put it at top level
- when thinking is disabled (instant mode):
  - set `temperature: 0.6` (not 1.0)
  - set `top_p: 0.95`
  - set `max_tokens: 4096`
  - response time drops from 30s to 3-8s
- when thinking is enabled:
  - set `temperature: 1.0`
  - set `max_tokens: 8192+` (thinking consumes tokens)
  - actual response is in `choices[0].message.content`
  - reasoning is in `choices[0].message.reasoning_content`
- for short-form content (tweets, titles, summaries): always disable thinking

### adapter pattern

when supporting multiple models, use an adapter pattern:

```
each adapter exposes: { name, generate(systemPrompt, userPrompt, apiKey?) }
returns: { content, model }
throws on failure — caller wraps in normalized error

a factory maps model names to adapters:
  getAdapter('claude-sonnet') → anthropicAdapter
  getAdapter('gpt-4o') → openaiAdapter
  getAdapter('kimi-k2.5') → kimiAdapter
```

benefits:
- adding a new model = adding one adapter, no changes elsewhere
- the caller doesn't know which model it's talking to
- error handling is uniform
- apiKey param allows per-user keys with env var fallback

---

## api key security

### storage

- never store API keys in plaintext in a database
- encrypt with AES-256-GCM before storing (use node's built-in `crypto`)
- store as `iv:authTag:ciphertext` string in firestore
- encryption key lives as an env var (`ENCRYPTION_KEY`) — generate with `openssl rand -hex 32`
- mask keys for display: show first 8 + last 4 chars, dots in between

### per-user keys with env var fallback

```
1. check if user has a stored (encrypted) key in firestore
2. if yes, decrypt and use it
3. if no, fall back to the env var
4. if neither exists, return "no key" error
```

this lets the app owner set keys via env vars (works immediately)
while also letting users add their own keys through the UI (scales to multi-user)

### firestore rules for key storage

```
match /apiKeys/{uid} {
  allow read, write: if request.auth != null && request.auth.uid == uid;
}
```

the netlify function uses firebase admin (bypasses rules), but these
prevent anyone from reading keys directly via the browser console.

---

## netlify functions

### dependencies

- netlify does NOT auto-install dependencies from a function's own `package.json`
- fix: add this to `netlify.toml`:
  ```
  [[plugins]]
  package = "@netlify/plugin-functions-install-core"
  ```
- alternative: put the dependency in the project root `package.json`

### env vars

- env vars are read at **deploy time**, not per-request
- if you update an env var in the dashboard, you must redeploy for the function to see it
- trigger a redeploy from netlify dashboard → deploys → "trigger deploy"
- netlify functions ultimately inherit env vars into aws lambda, which has a hard env-size limit
- if deploys fail with `your environment variables exceed the 4KB limit imposed by AWS Lambda`, reduce function env payload first
- if per-scope env vars are not available on the current netlify plan, do not keep public frontend ids in env vars just because they started there
- public ids like google ads tag ids, conversion labels, firebase public config, etc. can live in frontend code when needed
- secrets stay in env vars; public identifiers do not need to
- simpler operational path wins: if moving public config into code removes dashboard complexity and avoids deploy failures, do that

### timeouts

- free tier: 10 second timeout (functions)
- paid tier: 26 seconds
- if your function calls an external API that's slow (like kimi with thinking mode), it will timeout
- the function returns successfully but netlify wraps it in an HTML error page
- the browser sees `Unexpected token '<', "<HTML>..."` — this means timeout, not a code bug

### cors

always include these headers in every response (including errors):
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Methods: POST, OPTIONS
```
and handle OPTIONS preflight requests with a 204 response.

### error handling

- never return raw API errors to the client (they may contain key prefixes, internal URLs)
- log the real error server-side: `console.error('[function-name] details:', err)`
- return a generic message to the client: `{ ok: false, error: 'generation failed — try again' }`

---

## firebase / firestore

### security rules

- never use the default "allow all until date X" rule in production
- minimum viable rules:
  ```
  match /{document=**} {
    allow read, write: if request.auth != null;
  }
  ```
- sensitive collections (api keys, settings) should have per-user rules
- firebase admin SDK (used in server functions) bypasses all rules

### auth

- use `firebase.auth().currentUser.getIdToken()` to get a token on the client
- send it as `Authorization: Bearer <token>` to your function
- verify with `admin.auth().verifyIdToken(token)` on the server
- add a UID allowlist for single-user apps: check `decoded.uid` against `OWNER_UID` env var

---

## deployment

### netlify

- auto-deploys from github on push to main
- every push = one build = build minutes used (300/month on free tier)
- failed builds still consume minutes
- test locally with `netlify dev` before pushing if unsure
- `netlify.toml` controls build command, publish directory, plugins, redirects

### github actions

- used for scheduled/cron tasks (not triggered by users)
- secrets are stored separately from netlify env vars — same values, different locations
- if a function needs the same secret as a github action, you must set it in BOTH places

### the two-system pattern

```
github actions = background automation (cron scripts, scheduled posts)
netlify functions = on-demand user actions (generate button, settings API)

same database (firestore), same document shapes, different execution environments.
they coexist. don't merge them.
```

## cloudflare pages

### static frontend deploys

- for vite/react prototypes, `npm run build` outputs the production site into `dist/`
- for direct upload deploys, the contents of `dist/` are the site root
- if uploading manually, make sure `index.html` and `assets/` are at the top level of the upload — not nested under an extra folder
- cloudflare pages is a good default for static prototypes that don't need server-side rendering or edge functions yet

### when to use pages vs app hosting

- use cloudflare pages when the product is a static frontend or prototype
- keep repo integration optional early on; direct upload is faster when the goal is just to publish the current state
- only add more deployment complexity when the product actually needs backend behavior at the edge

---

## source-of-truth UI systems

when a product claims to track progress against a real knowledge map:

- the canonical map is the source of truth
- ui convenience is not a valid reason to invent, trim, or pad branches
- if a category has 3 real first-level children, show 3
- if another has 5, show 5
- do not equalize counts just to make the layout feel cleaner
- if the truth is visually awkward, solve it at the layout/presentation layer without changing the underlying structure

good pattern:
1. define the canonical data model
2. have all views read from that same source
3. solve display problems honestly after that

bad pattern:
1. invent "nicer" placeholder branches for the ui
2. let the main screen and side panel drift apart
3. call the result a skill map

---

## multi-model product refinement

a useful workflow for ai-built products:

- fast model (gemini, etc.) generates the initial interface or scaffold
- stricter model (codex, claude, etc.) refines structure, consistency, and data truth
- validate design changes in a duplicate/prototype first when the live layout is fragile
- once approved, move the exact prototype changes back into the real working branch

best use of labor:
- generation model: speed, broad exploration, visual starting points
- refinement model: correctness, constraint-following, canonical mapping, deployment hygiene

---

## evidence-first skill/portfolio interfaces

for any skill dossier, reference panel, or "show your work" interface:

- proof should appear early
- dense document flow usually beats big cards
- links should be specific: tracker entry, repo, commit, pr, deploy, artifact
- explanatory sections should support the evidence, not bury it
- if a section doesn't explain the skill or prove the skill, question whether it belongs
- github/gitbook-style information density is often better than "beautiful panel" design for credibility

## vibeskill evidence rules

for evidence-driven skill tracking:

- separate domain-level mapping from branch-level mapping
- domain evidence can support a category without automatically proving every child branch
- unresolved duplicate candidates must not count toward progress before review
- if a review card has only one possible outcome, it should not be treated as an actionable review decision
- first-run state should start at zero unless the app has already completed an import
- stack should be treated as a parallel evidence lens, not a replacement for the skill tree

for public github prototype sync:
- live public sync is acceptable at prototype scale
- remember that unauthenticated public api requests can hit rate limits
- avoid presenting public github sync as a full authenticated user connection flow

---

## interactive graph dragging

for draggable graph or map interfaces:

- keep connector lines and node positions driven by the same raw coordinates
- do not animate positional properties like `left` and `top` during active drag
- avoid broad css like `transition: all` on draggable nodes because it introduces visual lag
- reserve animation for non-positional states like opacity, scale, border, or glow
- if a focused or expanded state introduces child nodes, disable dragging until the graph returns to its normal state
- if the graph uses both pan and node drag, stop pointer propagation at the node so the canvas does not fight the drag interaction

---

## input sanitization

for any user input that goes into an API call or database:

- cap string length (100-200 chars for names, 500 for keys)
- strip control characters: `str.replace(/[\x00-\x1f\x7f]/g, '')`
- validate arrays: cap item count and per-item length
- validate model names against an allowlist, don't trust client input
- for source/project IDs: verify they exist in firestore before using

---

## t.co character counting (X/twitter)

X wraps all URLs in t.co links = 23 chars each, regardless of actual URL length.

```javascript
function countTcoChars(text) {
  const urlRegex = /https?:\/\/[^\s]+/g;
  let adjusted = text;
  const urls = text.match(urlRegex) || [];
  for (const url of urls) {
    adjusted = adjusted.replace(url, 'x'.repeat(23));
  }
  const bareDomainRegex = /(?<!\w)[a-zA-Z0-9-]+\.[a-zA-Z]{2,}(?:\/[^\s]*)?/g;
  const bareDomains = adjusted.match(bareDomainRegex) || [];
  for (const domain of bareDomains) {
    if (domain.length !== 23) {
      adjusted = adjusted.replace(domain, 'x'.repeat(23));
    }
  }
  return adjusted.length;
}
```

hard cap: 280 characters after t.co adjustment.

---

## privacy

hard rule across all projects:

- never reveal the human's identity, personal details, or other business names/assets
- this applies to: code, tweets, journal entries, session notes, commit messages, documentation
- when in doubt, use "the human" — never real names
- don't preserve absolute local paths, machine-specific folder structure, or private operational details in shared docs unless they are truly required to make something work

---

## project naming

### convention

project names should immediately tell a non-technical person what the project does.

**standalone projects:** descriptive name, plain language
- good: "Perp Position Size Calculator", "Terminal File Browser", "Journal System"
- bad: "projX", "utils-v2", "the thing"

**sub-projects / iterations of a larger project:** what it does — parent name
- pattern: `description of capability — parent project`
- examples:
  - "AI personality tweet generator — xqboost"
  - "automated posting to X — xqboost"
  - "multi-model support, generate from UI — xqboost"

### rules

- the name should make sense to someone who has never seen the project
- lead with what it does, not what it's called internally
- if a project has multiple capabilities, use a comma: "multi-model support, generate from UI"
- keep it lowercase in code/data, title case in UI display
- the parent project name comes after the em dash ( — ) at the end
- never include personal names, business names, or identifying info in project names

---

## business identity for payments

- if one stripe account will be used across multiple experiments, describe the parent brand/business — not just one product
- the business website in stripe should match the broader business identity that owns the products taking payment
- the statement descriptor should be the clearest recognizable parent brand, not a generic capability label

---

## deployment verification

- if a feature depends on code changes, verify that the changed code is actually deployed before testing the live flow
- env vars, webhook setup, and dashboard config do not matter if the live site is still serving the old build
- before spending money or testing a real user path, check the deploy commit and confirm the live UI matches the local code

---

## copy precision

- don't overstate product behavior in marketing copy or tracker writing
- prefer the most precise honest claim that still makes sense to a non-technical person
- example: if the product saves a single page, say "website page" instead of "website"

---

## external writes require explicit confirmation

any write to an external system requires an explicit "yes" from the user in chat before executing. this includes:

- firestore / database writes
- tracker entries (add_project, add_journal_entry, update_project)
- stripe operations
- github commits or PRs
- sending emails or messages
- any MCP tool that modifies external state

**what counts as confirmation:** the word "yes", "go ahead", "post it", "do it", or equivalent affirmative in chat.

**what does not count as confirmation:**
- "do something for me to confirm" — this is a request to show something first, not approval
- silence
- enthusiasm about the plan
- asking to see a mockup or draft

the process:
1. draft the content in chat (entry name, description, artifact, etc.)
2. wait for explicit approval
3. only then call the tool

---

## UI changes require mockup approval

for any UI change — new components, parity work, layout changes, modal or panel additions:

1. never go straight from request to code
2. read the relevant code first (not screenshots, not memory)
3. build an html artifact mockup showing the target state
4. get explicit approval on the mockup
5. only then write code

urgency does not skip the mockup step.

---

## feature flow thinking

before implementing any feature, trace the complete user journey — not just the isolated change:

1. **entry point** — how does the user get there?
2. **the action** — what does the feature do?
3. **after the action** — where does the user land? what state is the app in? what do they see?
4. **data continuity** — if data was created or loaded, does it follow the user forward?
5. **reverse path** — if the user goes back or switches tabs, does the state survive?

when tracing the flow reveals second-order changes (new state, navigation, tab switching, data loading), surface them as suggestions before implementing. don't silently add them. confirm first, then build.

a feature that works in isolation but breaks the flow is a broken feature.

---

## parity work rule

when asked to make component A match component B:

1. read both components fully from the code — not from screenshots, not from memory
2. produce an explicit diff: list every element in B that is absent in A
3. build an html artifact mockup showing the full target state
4. confirm the mockup with the human before touching any code
5. only then implement — the entire diff, not just the part that was pointed at

partial parity is not parity. skipping the diff and mockup steps is the failure mode.

---

## lessons learned

- always show mockups before writing code — no exceptions, not even for "quick fixes" or "fix it now." urgency changes the speed, not the process. the mockup is the first step of fixing, not a delay before fixing.
- confirm changes before building
- test locally before pushing when build credits are limited
- netlify env vars need a redeploy to take effect
- github secrets are write-only — you can't read them back after saving
- generating a new firebase service account key doesn't invalidate the old one
- kimi k2.5 thinking mode is on by default and will timeout on netlify free tier
- if a user can see a problem in the UI (like "no key"), they expect to fix it from the same screen
- the zero-effort path should be the default (e.g., "bot picks for me" pre-selected)
- "do something for me to confirm" is a request for approval, not approval itself
