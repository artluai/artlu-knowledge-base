# best-practices.md — shared knowledge base

reference doc for all projects. any claude instance should fetch this
before building anything that touches APIs, auth, deployment, or security.

last updated: 2026-05-01 (tracker title brevity rule; reframed design rules as "rules of thumb" / heuristics, not hard rules; sharpened #2 and #6 to be more meta; project identity vs doc identity, scope discipline on MCP writes, cwebp downscale, stop-on-wrong, distinguishing reasons)

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
- any openai-compatible API (OpenRouter, moonshot, together, etc.) uses the same shape — just swap `baseURL`

### openrouter

- endpoint: `https://openrouter.ai/api/v1/chat/completions`
- auth header: `Authorization: Bearer <key>`
- openai-compatible format — same shape as above
- preferred provider for Qwen models: `qwen/qwen-plus` (chat), `qwen/qwen-turbo` (scoring/fast)
- use over DashScope or MuleRouter — more reliable, broader model selection, same code works for any model swap

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

### cloudflare pages

- for vite/react prototypes, `npm run build` outputs the production site into `dist/`
- for direct upload deploys, the contents of `dist/` are the site root

### node version fallback for rspack / remotion issues

- current known machine state: Node 22 is the default on this machine because Node 24 caused repeated rspack / remotion native-binding failures across projects
- on this machine, some projects using remotion / rspack have failed under Node 24
- the failure can show up as `@rspack` native binding errors, `ERR_DLOPEN_FAILED`, or native module / code-signature load failures
- Node 22 is the first fallback when this happens
- if a project hits this class of error:
  1. switch to Node 22
  2. reinstall dependencies
  3. retry the build or render

preferred fix path:

```bash
nvm use 22
rm -rf node_modules package-lock.json
npm install
```

- if Node 22 fixes the problem, keep Node 22 as the default for that machine or shell setup
- treat this as a cross-project runtime compatibility issue, not a one-off repo quirk

---

## VPS projects (Hetzner)

### server setup (standard)

```bash
apt update && apt upgrade -y && apt install -y curl nginx docker.io
systemctl enable docker && systemctl start docker
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs
npm install -g pm2
pm2 startup
```

### postgres via docker

```bash
docker run -d --name postgres --restart always \
  -e POSTGRES_PASSWORD=yourpassword \
  -e POSTGRES_DB=yourdb \
  -p 5432:5432 postgres:16
```

### nginx proxy pattern

```nginx
server {
    listen 80;
    server_name _;
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### deploy workflow

```bash
# after GitHub Desktop push:
cd /bots/[project] && git pull && pm2 restart [project]

# if .env changed:
pm2 restart [project] --update-env
```

### playwright on VPS

- install deps first: `npx playwright install-deps chromium`
- then browser: `npx playwright install chromium`
- this is a large download (~300MB) — it will appear to hang, give it 5 minutes
- `npm install playwright` runs instantly if already installed — check with that first

### ARM vs x86 on Hetzner

- CAX11 (ARM) is cheapest but has Playwright quirks
- CX23 (x86 AMD) at ~$6/mo is recommended for Node.js bots that use Playwright
- x86 has better npm ecosystem compatibility

---

## tracker conventions

### project name format

```
[what it is/does] - [Project Name]
```

description first, project name last. always.

**good:**
```
AI agent control panel mockup - Animabot
keyword filtering feature, stripe billing - Pipelinecpc
reddit brand monitor
interactive skill-tree visual mockup — VibeSkill
```

**bad:**
```
Animabot - AI agent control panel mockup   ← name first, wrong
Animabot mockup                            ← too vague
Animabot                                   ← no context
```

if the project name is self-explanatory (e.g. "reddit brand monitor"), no dash needed.

**keep titles short and readable at a glance.** the title is the first thing a visitor sees in the project table — long titles get truncated, comma-stuffed titles read like changelog notes. cut to the smallest phrase that conveys the deliverable; move detail into `longDesc`.

**too long:**
```
subnet-client adapter, broadcast detection, invite evaluation — Animabot
multi-model support, generate from UI — xqboost
```

**tighter:**
```
subnet-client adapter — Animabot
on-demand tweet generator — xqboost
```

drop secondary capabilities, abbreviations, and conjunctions when they're not load-bearing. if you can't decide between two phrasings, pick the one that fits in a tweet.

### longDesc format

break into short **bold labelled sections**. write for two audiences: a developer and a non-technical person.

```
**What is [Project]?**
One or two sentences. Plain English.

**What it does**
Major features, screens, or flows. Name specific things.

**How it works** (optional, for technical entries)
Stack choices, architecture decisions.

**Built with**
Technologies. Short.
```

rules:
- direct and specific, no filler
- don't start with "This project is a..." or "I built this because..."
- short sentences, one idea per line
- technical terms OK — but explain the purpose, not just the name

### longDesc format — videos (different from code projects)

video project entries (spoolcast videos, news-anime-bot episodes, any video deliverable) use a different longDesc shape from the bold-labelled-sections pattern above. viewers visit the tracker page to watch the video, not to read about the pipeline — so lead with content, end with a brief build note.

structure:

1. **the content first.** write the lead like the youtube/tiktok description a viewer would read. story summary, satire copy, or whatever the video is actually about — in the show's editorial voice. no `**What is X?**` label up top; just the content prose. this is what earns attention on the embed below it.
2. **`**Sources**`** as a short bullet list when the video makes factual claims. every real number, date, and named entity should be traceable.
3. **AI disclosure** if the video is AI-generated. one short paragraph. tune wording to whether real-figure caricatures are present. avoid `parody / not real` framings when the underlying news is real and sourced — that's editorially dishonest.
4. **`**How it's made**`** as the closing label. one paragraph: stack, rough cost, "built on the spoolcast engine" or similar. brief.

what does NOT belong in a video tracker entry: detailed build logs, post-mortems, what-changed-vs-previous-episode rundowns. those live in the session dir's `script.md`, the show's `rules.md`, or `DESIGN_NOTES.md`. if a "what changed" section is creeping into the longDesc, move it.

### media field — required for videos

`media` must be the youtube `embed/<id>` URL (e.g. `https://www.youtube.com/embed/uBfr45P2aOM`). the tracker frontend renders this as an iframe — without it the project page shows no inline player. don't skip; the iframe is the whole point of a video entry.

screen.studio links go in `link`, not `media` (screen.studio blocks iframes — see `## Video Embeds` in `artluai-tracker/CLAUDE.md`).

### name format — series episodes

for recurring series (news-anime-bot episodes, dev-log series, etc.), use `<series name> ep N — <topic>`:

**good:**
```
daily ai news satire ep 2 — microsoft's $120b ai bet
spoolcast dev-log #3 — caption burn-in pipeline
```

the series name + episode number give continuity; the em-dash topic gives specificity. the existing project-name-format rule (description first, name last) still applies for one-off projects.

### artifactHtml field

paste raw HTML directly to embed a live demo via `iframe srcdoc`. no deploy needed. the site renders it automatically. use this for mockups, interactive tools, any single-file HTML artifact.

---

## project documentation standards

every serious project should have these three files in the repo root:

### README.md
for GitHub visitors. what it is, how to set it up, environment variables, architecture overview. written for someone who wants to fork and run it.

### CLAUDE.md
for Claude. full technical context to resume work in a new session without re-explaining. includes:
- server details (IP, SSH, directory)
- complete file structure with what each file does
- database schema (all tables, all columns, purpose)
- all environment variables
- API routes (public and admin)
- current status (what works, what's pending)
- deploy workflow
- pending items

### PRD.md
product requirements. the vision, current state, planned features, design principles, technical constraints, out of scope. the north star doc. write it so someone new to the project understands where it's going and why decisions were made.

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

## mechanical verification of fixes

"done" means a reproducible check ran against the final artifact and passed — not "code changed," not "intermediate data looks right," not "I spot-checked by eye."

- the check can be programmatic (a test, an audit script, a curl against the live URL) or manual (a specific click-path with a specific expected result), but it must be repeatable and run against the thing the user actually consumes
- code change = a fix attempt, not a fix. diagnostic models of bugs are always partial; a model that explains N of M reported symptoms doesn't mean the other M-N are gone — it means your model doesn't cover them
- the audit doesn't depend on the diagnosis being complete. it re-checks the artifact from scratch. run it before claiming a fix shipped, even when "I'm sure that was the right change"
- examples of the pattern in practice:
  - spoolcast: `audit_render.py` runs against the final mp4 and writes a sentinel file before anything is called shipped. the rule is "render isn't done until the audit passes" — applies in human-in-loop AND autonomous modes
  - deployment: verify the live build matches local code (see section above — this is one instance of the broader rule)
  - ui changes: the mockup approval + the final-state screenshot check are both mechanical verifications, not eyeballed "looks right"
- when a project has no check yet, define the check before claiming done. "what's the mechanical thing I'd run to prove this?" is the right first question
- failure mode this prevents: fix the mechanisms you diagnosed, declare done based on diagnostic closure, user points out the symptom is still there, repeat. the audit is the exit condition — not your confidence in the fix

---

## copy precision

- don't overstate product behavior in marketing copy or tracker writing
- prefer the most precise honest claim that still makes sense to a non-technical person
- example: if the product saves a single page, say "website page" instead of "website"

---

## project identity ≠ firestore doc identity

one conceptual project can map to multiple firestore docs. before acting on project-level fields (`showcase`, `top`, etc.), check whether the user's "one project" means one doc or several — grouping is often by URL or intent, not by row. example: VibeSkill and canonical skill-map are two docs pointing at the same `vibeskill-prototype.pages.dev` URL; unflagging one leaves the other visible.

---

## distinguishing reasons when picking from a set

when proposing a specific item from a set, lead with the reason that narrows the set — not a necessary filter shared by multiple candidates. "has `showcase: true` + untouched today" describes a pool; it doesn't pick one item out of the pool. if your reason applies equally to 3 candidates, it's a filter masquerading as a reason.

---

## stop on first "wrong"

when the user says a proposal is wrong, don't cycle through alternative answers. ask what's wrong with the approach or inputs. chained guesses after "wrong" compound the error and force the user to re-litigate the whole instruction.

---

## scope discipline on follow-on MCP writes

after executing a requested action, if the visible outcome doesn't match what the user would naturally expect (a project disappeared, a cascade fired, an item appeared), report the delta in plain english and wait. do not chain a corrective MCP write to paper over the side effect — it needs its own explicit authorization. "fixing" unasked-for side effects compounds the original miss.

---

## image downscale for committed assets

when committing images sourced externally (AI-generated scenes from kie.ai, fetched memes, etc.), downscale via `cwebp` before commit: `cwebp -q 72 -resize 640 0 in.png -o out.webp` typically yields ~60× size reduction with visually lossless results at thumbnail display sizes. ffmpeg's default brew build does NOT include libwebp encoder — install the standalone tool via `brew install webp`. originals stay in the source repo; the web-serving repo only holds the downscaled versions.

---

## artlu.ai design rules of thumb

heuristics pulled from past design feedback. useful as a starting point when there's no specific direction yet — these aren't hard rules. if the human pushes back on one, follow the push. but as default behaviors when shipping a first version, these tend to get closer to the answer than convention.

1. **every meaningful block tends to want a section header** — even if the block holds a single piece of content. if there's a "summary" head on the right, the video on the left usually wants one too ("youtube", "tiktok"), not floating without a label.
2. **adjacent elements tend to behave the same way.** alignment is the visible case (tops line up), but the pattern is broader: vertical rhythm (gap between sections), wrapping behavior (whether long text wraps or runs off), padding, sizing — siblings tend to match unless there's a reason. when something feels "off", look for the asymmetry between two related elements and equalize it.
3. **margins tend to feel symmetric** unless there's a reason. defaults like `padding: 48px 24px 80px` (top different from sides) usually read as wrong; equal top + sides reads as deliberate.
4. **above-the-fold matters for primary assets.** if a section holds a hero image or video, the full asset usually needs to fit in the viewport on first paint at common screen sizes. cap height with `min(calc(100vh - X), Y)` and let aspect-ratio derive the rest.
5. **format-aware UI, same design system.** different content shapes (long vs short, widescreen vs vertical) tend to want different layouts — but stay within the same colors, fonts, radii, and section conventions. rotate or reshuffle existing patterns, don't invent new ones.
6. **layout tends to follow content.** containers usually want to auto-fit their contents (a grid column with a derived-width item should be `auto`, not a hardcoded px). cards expand for longer descriptions before shrinking back. hardcoded numbers that don't account for content are usually where dead space comes from.
7. **brief feedback maps to specific fixes** — "still cut", "feels weird", "too wide", "looks bad" usually point to a concrete property (a height calc, a gap, a flex direction). first guess: is something not aligning, not fitting in viewport, or wasting space?

these are heuristics, not laws. when in doubt, ask. when tempted to add an 8th, sharpen one of these instead.

see the artluai-tracker `mockups/` directory for HTML reference implementations.

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

## approved artifact embedding

when embedding an approved artifact into a tracker entry or any external system:

- always read the file directly from disk — never reconstruct from memory
- the file that was presented and approved is the source of truth
- do not rewrite, compress, or re-encode the content before passing it
- unicode characters stay as-is — do not convert to escape sequences

correct: read file from disk → pass content directly to `update_project`
wrong: reconstruct HTML from memory → pass to `update_project`

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
5. **reverse path** — if the user goes back, switches tabs, reloads, signs out/in, imports, merges, or restores later, does the state still behave correctly?

when a feature creates a new data type, saved object, or mode, also trace its downstream workflow effects:

- how it is labeled in the UI
- where it reloads back into the app
- whether it routes to the correct screen or tab
- whether merge/import/export rules still make sense
- whether old saved data needs a fallback mapping
- whether selection, filtering, badges, and restore logic still reflect the correct type

when tracing the flow reveals second-order changes, surface them as suggestions before implementing. don't silently add them. confirm first, then build.

a feature that works in isolation but breaks the later workflow is a broken feature.

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
- never reconstruct an approved artifact from memory — read it from disk
- never paste large HTML files into the terminal — give them as downloads for GitHub Desktop
- on Hetzner: Playwright downloads silently with no progress bar — give it 5 minutes before assuming it's stuck
- x86 AMD over ARM for VPS when Playwright is involved — fewer compatibility edge cases
- `pm2 restart --update-env` is required when `.env` changes, plain restart doesn't pick up new vars
- HTML files served by Express must be in the `public/` subfolder — not the repo root
- code change is a fix attempt, not a fix — run the mechanical check against the final artifact before claiming shipped. skipping this was the recurring failure during spoolcast V4 debugging: diagnose 2 of 5 symptoms, mark batch complete, user watches the render, other 3 still there. burned multiple re-render cycles. the audit script is now the exit condition, not the agent's confidence
