# context.md — shared project context

read this first in any new chat before making suggestions, writing tracker content, or changing infra.

last updated: 2026-04-22 (video showcase live on artlu.ai)

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
- verified = mechanical check passed, not code changed. when fixing a reported issue, "done" means a reproducible check ran against the final artifact (rendered file, deployed app, shipped artifact) and passed. "I changed the code" / "the intermediate data looks right" / "I spot-checked by eye" are proxies, not verification. if the project has a check script (e.g. `audit_render.py` in spoolcast), running it on the artifact IS the bar. otherwise, define the check before claiming done. causal models of bugs are always partial; the mechanical audit doesn't depend on the diagnosis being complete.

## shared references

- `best-practices.md` = shared patterns for APIs, auth, deployment, security
- this file = high-level context and current state

## active systems

### animabot

- repo: https://github.com/artluai/animabot
- server: Hetzner VPS at 195.201.90.47
- bot directory: `/bots/animabot/`
- admin panel: http://195.201.90.47
- public panel: http://195.201.90.47/public.html
- stack: Node.js, matrix-js-sdk, ethers.js, OpenRouter (Qwen), Postgres, Express, PM2
- purpose: AI agent framework — bots that live in Matrix chatrooms with their own Ethereum wallet, evolving personality, and daily ego reflection
- first bot: Zara (ENFP) — fully live in abliterate.ai General and Subnet Meta rooms
- wallet: 0x66446a0A966390f786373568caE3F816500435ae
- platform adapters: Matrix/abliterate (built), Telegram/Discord (planned)
- current status: fully live — Matrix connected, avatar set, personality tuned, reflection cycle running
- abliterate uses ETH wallet auth via subnet-client npm — not standard Matrix password
- deploy: `git stash && git pull && pm2 restart animabot` (server often has local edits that block pull)
- proactive chime: off by default, toggled via admin panel, timer resets on enable, ignores pre-startup messages
- avatar: set manually via admin panel only — NOT on startup (causes "changed profile picture" spam)
- reflection: 3am daily, detects tone feedback, logs caused_by per entry
- chime_config in DB: {enabled, interval_ms, min_messages, probability}
- full technical reference: see `CLAUDE.md` in repo
- full PRD: see `PRD.md` in repo

### artlu.ai / tracker

- repo: `artluai-tracker`
- stack: react, vite, firebase, firestore, netlify
- firestore collections: `projects`, `journal`
- purpose: public tracker for `100 projects in 100 days.`
- journal voice: lowercase, short, honest, specific, non-corporate
- main local guidance: `rules.md`
- github import: now uses live public github repo sync
- duplicate candidates no longer count toward progress until reviewed
- branch mappings were tightened so child branches only receive explicit branch-level credit
- major remaining gap: ai session imports are still not real
- emerging product direction: add a stack-organized view/tree as a parallel lens for portfolio-facing use
- video showcase category now live — homepage 4×2 grid above demo showcase, `/video/:id` guidebook pages fed by `public/videos/<id>/bundle.json` synced from sibling `spoolcast-content/` via `scripts/sync-video.mjs` (requires `brew install webp` for image downscale)
- key interaction finding: draggable category nodes should not animate positional properties during drag; connector lines and boxes need to read from the same raw coordinates or the map feels laggy
- key interaction finding: once child branches are expanded, dragging should be disabled until the user collapses back to the normal system view
- likely next correction: first-run state should begin at zero and only fill after import

### artlu tracker mcp

- repo: `artlu-tracker-mcp`
- purpose: mcp bridge for reading and writing tracker projects + journal entries
- tool surface: `add_project`, `list_projects`, `get_project`, `update_project`, `add_journal_entry`, `list_journal_entries`, `get_journal_entry`, `update_journal_entry`, `delete_journal_entry`, `list_projects_with_ids`
- codex setup now works with the same local node server and firestore credentials previously used in claude
- a real tracker backup exists in the mcp repo `backups/` folder
- known quirk: mcp-created tracker projects need a manual `slug` update after create
- observed in this codex session: `tags` did not persist on `add_project` and needed a follow-up `update_project`

### spoolcast

- repo: https://github.com/artluai/spoolcast (code / rules / scripts / Remotion composition)
- content dir: sibling directory `spoolcast-content/` (per-session data, media, shot-lists, renders — kept out of the repo so one clone drives many videos)
- domain: spoolcast.com
- stack: Claude Code (or any coding agent), Remotion, kie.ai (nano-banana-2 default), Google Cloud TTS (ElevenLabs adapter coming), Python 3.14, Node 22, ffmpeg (homebrew-ffmpeg tap required for libass subtitle burn-in)
- purpose: turn chat logs, docs, or ideas into narrated illustrated videos. coding agent drives editorial stages via rule files; scripts drive production (image gen → TTS → preprocessor → render → audit → publish)
- rule files (read in order): `rules.md` (index + non-negotiables + general agent rules) → `PIPELINE.md` (workflow stages, session config, shot-list schema) → `STORY.md` (script extraction + pacing) → `VISUALS.md` (asset generation, preprocessor, transitions) → `SHIPPING.md` (review + publish)
- `DESIGN_NOTES.md` = why-log (what was tried + killed); `ROADMAP.md` = backlog for future work
- current transition vocabulary: `cut` + `crossfade` only. paint-on is deferred until the preprocessor outputs RGBA frames (current stroke_reveal emits RGB-with-white which flashes at every entrance). fade-to-white as a standalone exit is banned. meme/broll/reuse chunks hard-cut on BOTH sides (not just into them)
- style library at `spoolcast-content/styles/<style-name>/` with anchor image + character/object references; per-session style locked via `session.json`'s `style` field
- render audit gate: `scripts/audit_render.py` must pass against the final mp4 before anything claims "shipped". writes sentinel at `working/render-audit.passed`. same rule in human-in-loop + autonomous modes
- shipped videos: pilot (Meta TRIBE explainer, ~5min), explainer (*I don't make videos. My AI pipeline does.*, ~8min, artlu.ai/project/what-is-spoolcast), dev-log #1 (*Building with AI: how I stopped my AI from silently breaking rules*, ~4min, youtu.be/i3Z480n1k6k)
- tracker projects: each video is its own tracker entry; `spoolcast — AI-to-video pipeline` (doc `o9431zuYldMh0P0NS6If`) is the meta-project covering the tooling itself
- caption convention: SRT includes both narration cues AND bracketed `[on-screen: …]` cues for text rendered on frames. mobile burn-in path strips the bracketed cues (redundant with the frame) via `--exclude-onscreen-cues`
- thumbnail convention: kie.ai outputs at 1376x768 which YouTube letterboxes — always rescale to exact 1920x1080 before upload (`ffmpeg -vf "scale=1920:1080:flags=lanczos"`)
- acts/bumpers: never use function-name labels ("PAYOFF", "THE HOOK", "THE REVEAL") — exposes craft to viewer. Use content-describing labels ("CONCLUSION", "THE INCIDENT", "THE DIAGNOSIS")
- per-video AI budget is a generation count (not dollars) set via `session.json`'s `ai_budget`. ~60 gens ≈ $3-5 at current kie.ai pricing
- shipped videos get per-video guidebook pages at `artlu.ai/video/<session-id>` — sync script lives in the artluai-tracker repo, reads spoolcast-content directly
- two roadmap items open: mobile export from widescreen (9:16/1:1 crop + burned captions); text-to-video for prompt-only symbolic chunks. handoff kickoff at `/Users/ralphxu/Documents/Projects/spoolcast-mobile-export-kickoff.txt`

### snapshot

- public app: `https://sitesnapshot.org`
- current architecture: netlify control plane, firestore job tracking, cloud tasks queueing, cloud run worker, firebase storage artifacts
- main result from the latest rebuild: url mode works end-to-end through the queued worker path
- screenshot mode improved through the same backend path, but still beta
- stripe is now live and working end-to-end
- current pricing: starter = 4 credits / 9.99, pro = 15 credits / 29.99
- free flow permissions bug was fixed by allowing users to create their own doc and update only `freeUsedToday`
- landing copy now says `website page` instead of `website`
- sign-in modal was softened to a calmer action-first version
- current direction: monetization is live; acquisition setup is now live too
- first google search campaign is published and currently in learning
- advertiser verification was completed in google ads
- purchase and begin-checkout conversions were created and wired
- google ads tracking ids were moved into frontend code after netlify function deploys hit the AWS Lambda env var size limit
- next focus: watch search terms, add negatives, and tighten the campaign only after real impressions arrive

### vellumray

- public site: `https://vellumray.com`
- purpose: business-facing brand site for traffic, support, payments, and future product experiments
- deployed on cloudflare pages
- `hello@vellumray.com` and `support@vellumray.com` forward through cloudflare email routing
- used as the parent business identity for stripe, not just one product

### xqboost

- tweet / automation system is active
- project names can feed announcement workflows, so titles should stay clear and human-readable
- prefer plain separators like commas, `+`, or words over awkward punctuation if a title may get reused in tweets

### vibeskill

- public prototype: `https://vibeskill-prototype.pages.dev/`
- current state: deployed public prototype
- purpose: map ai-assisted coding work onto a structured software engineering skill tree
- current workflow: gemini-generated frontend scaffolding refined by codex into a more faithful canonical skill-map prototype
- important rule: the canonical skill map is the source of truth; ui should not invent or smooth branch structure for presentation
- important rule: the approved main neural-map screen is effectively frozen for now; layout changes should be tested in a duplicate/prototype first
- current improvement areas: move approved prototype refinements back into the working branch, keep sidebars/evidence views dense and reference-heavy, and continue replacing placeholder/demo content with canonical content
- deployment: static frontend on cloudflare pages
- tracker import: now backed by a real tracker snapshot instead of seeded placeholder records

### pipelinecpc

- public app: `https://pipelinecpc.com`
- repo: `artluai/pipelinecpc` (private)
- purpose: saas version of the keyword pipeline local tool
- stack: node.js, vanilla js, firebase auth, firestore, stripe, railway
- architecture: single index.html + server.js, deployed on railway
- auth: firebase google sign-in, token verified server-side on every request
- billing: $29/month subscription (stripe), credit packs on top ($10/$25/$50), 250 free keywords on signup
- credits deducted per keyword returned from the api
- free users: 21 major categories only, no category browser
- pro users: full 3,000+ category browser + credit top-ups
- key finding: dataforseo `keywords_for_categories` does not support server-side filters — all filtering is local
- firebase admin credential: must use full service account json as one env var (`FIREBASE_SERVICE_ACCOUNT_JSON`), not split into three separate vars — splitting causes firestore auth to fail even though token verification works

## naming and writing rules

- project names should say what the thing does
- for sub-projects or iterations: `capability — parent project`
- lead with the result, not the internal codename
- avoid sounding overly technical just to sound smart
- never doxx the human or reveal business names, business assets, private identifiers, or personal details in code, docs, tracker entries, journal entries, tweets, or other content

## current priorities

- pipelinecpc: google ads conversion tracking and landing page copy
- pipelinecpc: stripe customer portal for self-serve cancellations
- pipelinecpc: keyword_ideas mode (seed keyword → server-side filtered results)
- snapshot: watch search terms, add negatives, tighten the campaign after real impressions
- vibeskill: port the approved prototype changes back into `radial-v2`
- animabot: Telegram adapter
- animabot: MBTI editor in admin panel
- spoolcast: mobile export from widescreen (A.1) — 9:16/1:1 crop, burned captions via libass, regen for unsafe chunks
- spoolcast: text-to-video for prompt-only symbolic chunks (bumpers, mood scenes) via kie.ai video models — exploration, not commitment
- spoolcast: ElevenLabs TTS adapter (currently Google Cloud TTS only; README flagged as coming-soon)
- keep tracker entries and journal entries current as major milestones ship
- update this file when a workflow, architecture, or source-of-truth setup changes
