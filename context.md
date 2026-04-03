# context.md — shared project context

read this first in any new chat before making suggestions, writing tracker content, or changing infra.

last updated: 2026-04-03

---

## who the human is

- one person shipping fast with ai
- no traditional coding background
- scope gets decided quickly
- product instinct is strong. the first ai draft is often too generic.
- plain language beats jargon
- momentum matters more than over-analysis

## how to work

- read repo-local instructions first. in this setup that usually means `rules.md` and `personality.md`, plus any repo-specific docs that actually exist.
- if something is important across chats, write it down in docs instead of trusting chat memory
- never doxx the human or reveal business names, business assets, private identifiers, or other identifying details unless explicitly needed for a local operational setup task
- shared docs should avoid absolute local paths unless they are truly needed to make something work
- do not invent session details, quotes, or "funny moments"
- check the real data before writing tracker or journal content
- keep explanations brief and practical
- for fragile visual systems, use prototype-first workflow: validate in a duplicate, then port the approved version back into the real branch
- if live code exists, use that as the visual source of truth instead of reconstructing from screenshots
- for skill-mapping systems, never change the underlying branch structure just to make the interface look nicer
- when building panels meant for external review, optimize for proof and clarity over flavor text

## shared references

- `best-practices.md` = shared patterns for APIs, auth, deployment, security, and tool-specific patterns
- `personality.md` = voice and style guide for journal entries and tracker content
- this file = high-level context and current state

## active systems

### artlu.ai / tracker

- repo: `artluai-tracker`
- stack: react, vite, firebase, firestore, netlify
- firestore collections: `projects`, `journal`
- purpose: public tracker for 100 projects in 100 days
- journal voice: lowercase, short, honest, specific, non-corporate
- main local guidance: `rules.md`

### artlu tracker mcp

- repo: `artlu-tracker-mcp`
- purpose: mcp bridge for reading and writing tracker projects + journal entries
- tool surface: `add_project`, `list_projects`, `get_project`, `update_project`, `add_journal_entry`, `list_journal_entries`, `get_journal_entry`, `update_journal_entry`, `delete_journal_entry`, `list_projects_with_ids`
- known quirk: mcp-created tracker projects need a manual slug update after create

### snapshot

- public app: `https://sitesnapshot.org`
- current architecture: netlify control plane, firestore job tracking, cloud tasks queueing, cloud run worker, firebase storage artifacts
- stripe is live, pricing: starter = 4 credits / 9.99, pro = 15 credits / 29.99
- current direction: monetization live, acquisition setup live, first google ads campaign published

### keyword pipeline tool

- local project: `~/Documents/projects/niche-research/`
- repo: `https://github.com/artluai/niche-research`
- tracker entry: "Keyword Pipeline — niche research tool"
- files needed: `index.html` (full app) + `server.js` (CORS proxy) — nothing else
- run: `node server.js` then open `http://localhost:3000`
- stack: vanilla JS, single HTML file, no framework, no build step
- data API: DataForSEO (`api.dataforseo.com`) — auth stored in localStorage as `kw_auth`, entered via API Key bar in UI
- **API key is NOT hardcoded in source files** — user enters it in the UI, it persists to localStorage

#### what it does

- fetches keywords from DataForSEO by category code(s) across 3,182 categories
- categories classified as digital (d) / physical (p) / both (b) — stored in localStorage as `kw_all_cats`
- visual drag-and-drop pipeline: filters above Run bar = server-side (billed), below = local (free)
- filters: CPC range, competition range, intent, volume, negative keywords
- results table: keyword, CPC, competition, volume, $/conv, conv/day, intent, trends link
- expanded modal view with search, filter bar (CPC, $/conv min/max, volume ≥, intent checkboxes), neg keyword bar, export CSV
- category browser: 3,182 categories, filterable by d/p/b, click badge to reclassify
- saved tab: keyword searches, negative keyword lists, category selections, monitor list (★)
- landing screen: "new pipeline" or "load saved setup" — shows when pipeline has no categories loaded

#### key formulas

- **$/conv** (est. cost per conversion) = `CPC ÷ CVR` — lower is better
  - color coded: green <$20, orange <$50, purple <$150, red $150+
- **conv/day** = `(volume/30) × CTR × CVR` — estimated daily conversion ceiling
- industry defaults: General CTR 3%/CVR 2%, Finance 4%/5%, Health 3%/3%, eCommerce 4.5%/3%, Legal 2.5%/4%, Education 3%/2.5%, Travel 3.5%/2%, Real Estate 3.5%/2%

#### localStorage keys

- `kw_auth` — saved API key (Basic base64)
- `kw_all_cats` — full 3,182 category list with d/p/b classification
- `kw_cat_selections` — saved category selections
- `kw_saved` — saved keyword searches
- `kw_neg_lists` — saved negative keyword lists
- `kw_monitor` — starred/monitored keywords

#### category files (import via Saved tab → Category selections → Import)

- `catsel-broad-600.json` — 630 digital categories, broad sweep
- `catsel-curated-200.json` — 180 curated best opportunities
- `catsel-3niches-90.json` — 70 categories: health/diet + PPC/marketing + writing/career/education
- `both-categories-reclassified.json` — reclassified types for 2,216 "both" categories (import via browser console script)

#### research findings (2026-04-03)

- ran 368 categories at 500/page, $1-9 CPC, comp 0.1-0.5, trans+comm, vol ≥10k → 43,667 keywords, $8.05
- top opportunities by data: myers briggs tests ($1.65 CPC, 1.4M vol), contract templates ($5.91, 301k), screen recording software ($8.04, 301k), wireframing tools ($8.10, 301k), PMP certification ($5.94, 165k)
- recommended product angles: career assessment tool (quiz → report → membership), niche diet meal plan subscription, PPC/keyword tool (this pipeline itself)
- $10-15 CPC range = $500-750 est. $/conv at 2% CVR — only viable for high-price products

## naming and writing rules

- project names should say what the thing does
- for sub-projects or iterations: `capability — parent project`
- lead with the result, not the internal codename
- avoid sounding overly technical just to sound smart
- never doxx the human or reveal business names, business assets, private identifiers, or personal details

## current priorities

- keyword pipeline: continue research runs with curated category selections
- keyword pipeline: evaluate myers briggs / career assessment product angle; tomorrow = signups + public use
- artlu.ai tracker: keep entries current as milestones ship
- update this file when a workflow, architecture, or source-of-truth setup changes
