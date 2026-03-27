# context.md — shared project context

read this first in any new chat before making suggestions, writing tracker content, or changing infra.

last updated: 2026-03-27

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
- do not invent session details, quotes, or "funny moments"
- check the real data before writing tracker or journal content
- keep explanations brief and practical

## shared references

- `best-practices.md` = shared patterns for APIs, auth, deployment, security
- this file = high-level context and current state

## active systems

### artlu.ai / tracker

- repo: `artluai-tracker`
- stack: react, vite, firebase, firestore, netlify
- firestore collections: `projects`, `journal`
- purpose: public tracker for `100 projects. 100 days.`
- journal voice: lowercase, short, honest, specific, non-corporate
- main local guidance: `rules.md`

### artlu tracker mcp

- repo: `artlu-tracker-mcp`
- purpose: mcp bridge for reading and writing tracker projects + journal entries
- tool surface: `add_project`, `list_projects`, `get_project`, `update_project`, `add_journal_entry`, `list_journal_entries`, `get_journal_entry`, `update_journal_entry`, `delete_journal_entry`, `list_projects_with_ids`
- codex setup now works with the same local node server and firestore credentials previously used in claude
- a real tracker backup exists in the mcp repo `backups/` folder
- known quirk: mcp-created tracker projects need a manual `slug` update after create
- observed in this codex session: `tags` did not persist on `add_project` and needed a follow-up `update_project`

### snapshot

- public app: `https://sitesnapshot.org`
- current architecture: netlify control plane, firestore job tracking, cloud tasks queueing, cloud run worker, firebase storage artifacts
- main result from the latest rebuild: url mode works end-to-end through the queued worker path
- screenshot mode improved through the same backend path, but still beta
- old problem: too much work was trapped inside one fragile request
- current direction: durable jobs first, then monetization and acquisition

### xqboost

- tweet / automation system is active
- project names can feed announcement workflows, so titles should stay clear and human-readable
- prefer plain separators like commas, `+`, or words over awkward punctuation if a title may get reused in tweets

## naming and writing rules

- project names should say what the thing does
- for sub-projects or iterations: `capability — parent project`
- lead with the result, not the internal codename
- avoid sounding overly technical just to sound smart
- never doxx the human or reveal business names, business assets, private identifiers, or personal details in code, docs, tracker entries, journal entries, tweets, or other content

## current priorities

- snapshot: add stripe
- snapshot: get the site ready for google ads
- keep tracker entries and journal entries current as major milestones ship
- update this file when a workflow, architecture, or source-of-truth setup changes
