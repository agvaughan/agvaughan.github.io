# EVERYTHING — Family Everything-App: Master Plan & Spec

## Context

Alex wants a single self-owned "everything app" for his family: it ingests his email, calendar, texts, and news; owns his todo list; is his **second brain** (Paperpile-like paper library + notes + links with unified search); launches and maintains **compute jobs** (the sailing CFD simulator on Modal); hosts kid-facing educational apps for O (9, soccer theme) and N (12, cute-animals theme); and is operated/extended by a persistent, safety-hardened LLM agent that can modify the app itself from natural-language prompts. Three interfaces (iPhone, iPad, Mac, web), 2-3 iOS devices, offline-capable, everything reversible.

This plan was produced from ~10 research subagent reports (mid-2026 verified) plus three rounds of user Q&A. It is the "spec of specs": each sub-app below gets a full SPEC.md executable solo by an Opus 4.8 agent; this document is the orchestrator's ground truth.

**Target repo: `github.com/agvaughan/everything`** (exists, private, empty-ish). This planning session ran in `agvaughan.github.io` — at execution time, add `agvaughan/everything` to the session via `add_repo` and do ALL work there (user has authorized this repo explicitly). Never touch repos not owned by agvaughan. If a branch is needed in this planning repo, only plan docs go there; all product code goes to `everything`.

## Decisions locked (from user Q&A — do not relitigate)

| Topic | Decision |
|---|---|
| Hosting | Cloud VPS **or** Mac mini — both approved; hardened (2FA, minimal surface). Recommended: small VPS (Hetzner-class) for the main stack + always-on Mac at home as iMessage bridge. All-on-Mac-mini is an acceptable variant. |
| Network | **Tailscale-private.** App reachable only on the tailnet; Tailscale app on each family device. Zero public attack surface. |
| iMessage | Always-on Mac available → BlueBubbles bridge, read + send. |
| Apple | Will pay $99/yr. Strategy: PWA-first, Capacitor wrap via TestFlight in a later phase. |
| Todos | App **owns** todos natively (no Todoist/Things sync). |
| Kids | O = 9 (soccer theme), N = 12 (cute-animals theme). **Math starts at 3rd-grade level for both**; plan 2-3 years of growth; LLM generates content for higher levels as they progress. |
| Chess | TWO apps: kid version (progressive teacher) AND **adult coach for Alex** — openings/middlegame *strategy and thinking frameworks*, not memorization; goal: 1900-on-puzzles → comparable real-game strength. |
| Email autonomy | **Approve-to-send in app**, plus: agent may auto-send *follow-ups* (replies in a thread where Alex already approved a prior send to the same recipients) via delay-queue with cancel window. |
| Twitter/X | **Logged-in browser session** (Playwright + persistent authenticated profile on the server) scraping Alex's own home feed ("Following" tab). Gentle cadence to avoid account flags. Nitter-RSS adapter as fallback. |
| API budget | **$100-300/mo hard cap**, enforced by daemon with usage tracking. Sonnet/Opus for user-facing work, Haiku for bulk classification. |
| Books | Free sources (Gutenberg/Standard Ebooks/StoryWeaver) + **sideloaded owned EPUBs** + **LibriVox audiobooks/read-along**. |
| First demo | **Shell + todos + one kid app** (Life sandbox), then email/news. |
| Admin gamification | Streaks & stats + progress-toward-goals with goal rollups, **including a goal-setting debrief**. Persona: "executive coach with good vibes." |
| Second brain | Core feature. **One-time Paperpile export import** (BibTeX/JSON + PDFs); the app then becomes the system of record for papers + notes + links. |
| Compute jobs | **Modal only** (no adapter abstraction until a second backend exists); target workload: `agvaughan/modal_cfd` sailing CFD. Agent has **budget-capped autonomy** to launch runs — free within a monthly compute cap, approval required beyond it. |

## Architecture (settled by research)

### Client
- **Installed PWA** (React + Vite + TS): same app on iPhone/iPad (Add to Home Screen) and Mac (Safari Add to Dock / Chrome). iOS installed PWAs now have reliable Web Push (Declarative Web Push, iOS 18.4+), are exempt from the 7-day storage eviction, and get large OPFS/IndexedDB quotas.
- **`packages/platform` abstraction** for notifications/TTS/storage/share so a **Capacitor wrap (TestFlight)** can swap implementations in Phase 5 without touching sub-apps.

### Server (single docker-compose, on the tailnet)
- **Hono + Node TS monolith**, Postgres, **pg-boss** (jobs/cron), **Better Auth** (device session + Netflix-style profile picker; kid PINs — PIN is a family-courtesy boundary, NOT a security boundary; the device session + tailnet is the security boundary), `web-push` (VAPID), Caddy for TLS on the tailnet.
- **Sync: self-hosted PowerSync** — Postgres → client SQLite (wasm/OPFS); client writes replay through the server API (server = single enforcement point regardless of what agent-generated clients do). **Timeboxed go/no-go spike in Phase 0**; pre-written fallback: hand-rolled delta sync (~1-2k LOC, per-row versions, LWW).
- **Ingestion**:
  - Gmail: own GCP OAuth client, **"Production + unverified"** mode (NOT testing mode — 7-day token trap), `gmail.modify` scope, `history.list` polling (1-5 min). Never delete — label/archive only.
  - Calendar: same OAuth client, `calendar` scope, syncToken polling.
  - News: **Miniflux + RSS-Bridge** containers as fetch layer (HN via Algolia API, Substack free pubs via RSS, paid Substacks via **email ingestion** through the Gmail pipeline) → webhook → normalized item store. X via the logged-in-browser scraper (own container, persistent Chromium profile; one-time login by Alex via tailnet-exposed VNC or cookie import; polls ~every 30-60 min with jitter).
  - iMessage: **BlueBubbles** on the always-on Mac (on the tailnet) → REST/webhooks to server. Keep bridge Mac on older macOS if Private API features wanted; treat as a degradable capability with health-check alerts.
- **Knowledge core (`server/knowledge/`) — a PLATFORM SERVICE, not a sub-app.** Rationale: papers, notes, saved links, highlights, email attachments, and agent memory all want the same primitives, and multiple sub-apps + the agent consume them. One unified store:
  - `knowledge_items` (typed: paper | note | link | highlight | file), `knowledge_blobs` (PDFs/EPUBs on server disk, content-addressed, backed up with Postgres), tags/collections, item↔item links (backlinks).
  - **Search**: Postgres FTS (tsvector, per-item + extracted PDF text via `pdftotext` at ingest) + **pgvector embeddings** for semantic search; one hybrid search API (`/knowledge/search`) used by the brain sub-app UI, the shell's universal search, AND the agent's recall tools.
  - **Metadata resolvers**: DOI (Crossref), arXiv API, URL unfurl — run server-side at ingest; BibTeX import/export.
  - **Paperpile migration job**: one-time importer for Paperpile's export (BibTeX + folder structure + PDFs); idempotent, journaled, dry-run mode first.
  - Consumers: `subapps/brain` (UI), link-intake (routes "add this" captures into it), news (save-to-brain), agent (quarantined summarizer writes summaries back as linked notes).
- **Compute jobs (`server/compute/`) — thin Modal integration, UI as a sub-app.** Modal-only per decision: `modal run`/lookup via Modal's Python client in a small sidecar container (Modal has no first-class TS SDK; the sidecar exposes launch/status/logs/artifacts over an internal HTTP interface), job rows in Postgres (`compute_runs`: params, state machine queued→running→succeeded/failed, cost actuals from Modal usage API), artifacts pulled to blob store (plots/VTK/CSV). The CFD code stays in `agvaughan/modal_cfd`; the everything-app stores *references* (git SHA + entrypoint + params JSON schema per registered job type). **Agent autonomy = budget-capped**: PreToolUse policy hook checks monthly compute spend against the cap — under cap: auto-approved (logged + notified), over cap: queued for approval. Kill switch pauses all launches.

### Agent runtime ("the house brain")
- **Claude Agent SDK (TypeScript) daemon** + pg-boss cron sessions. NOT OpenClaw (its 2026 security record — one-click-RCE CVE, malicious-skill marketplace, exposed gateways — is disqualifying for an email-reading agent). We adopt OpenClaw's good patterns instead:
  - **Memory**: `MEMORY.md` (curated, ~2-4k token budget) + `memory/YYYY-MM-DD.md` daily notes, all in git; pre-compaction flush hook; grep-based recall (no vector DB unless recall demonstrably fails).
  - **Overnight dream job**: nightly session reviews the day, consolidates memory, writes `DREAMS.md` entry, proposes self-improvements **as PRs only — never direct commits**.
  - Morning brief job (triage summary, todo nudges, news digest) delivered via push + in-app.
- **Safety stack** (architecture over classifiers):
  1. **Quarantine pattern (the core defense)**: a no-tool subagent reads all untrusted content (email bodies, news items, fetched links, texts) and returns only schema-validated structured summaries; the privileged agent never sees raw untrusted text.
  2. **`anthropic-experimental/sandbox-runtime` (srt)** wraps all shell exec: filesystem + network isolation with an **egress allowlist** (Anthropic API, Google APIs, git remote — nothing else by default).
  3. **PreToolUse hooks = deterministic policy engine**: email send, any delete-equivalent, purchases, push-to-main → require human approval (`canUseTool` flow). Hooks are code, not model judgment.
  4. **Append-only JSONL audit log** of every tool call (PostToolUse hook).
  5. Email tools are label/draft-only; sends go through **approval + delay queue** (idempotency keys; cancel window; the follow-up auto-send exception per policy above).
  6. Red-team prompt-injection fixtures run in CI against the quarantine pipeline.
- **Budget enforcement**: per-session usage recorded; monthly hard cap $300 → daemon degrades to essential jobs only + alerts Alex.

### Backup & rollback (layered, everything reversible)
1. **Code**: git; agent changes = branch + PR, human merges. CI green required.
2. **Data**: Postgres WAL archiving (pgBackRest/wal-g) → point-in-time recovery + nightly snapshots; monthly **automated restore drill**. Append-only **action journal** with inverse ops for every mutating operation; soft-delete + tombstones only (no hard-delete tool exists).
3. **External side effects**: never-delete email policy; approval gates + delay-queue with cancel window for sends; anything financial = synchronous approval always.
4. **Memory**: markdown in git → `git revert`.

## Monorepo layout (`agvaughan/everything`)

```
everything/
├── AGENTS.md                 # global agent rules; CLAUDE.md symlinks to it
├── README.md  ROADMAP.md     # ROADMAP = phase/WP dispatch board + per-milestone demo scripts
├── package.json  pnpm-workspace.yaml  turbo.json  tsconfig.base.json
├── eslint.config.mjs  .dependency-cruiser.cjs   # mechanical boundary enforcement
├── docker-compose.yml  docker-compose.dev.yml  .env.example
├── .github/workflows/        # ci.yml, e2e-full.yml, deploy.yml; PR template requires spec link + AC→test map
├── apps/pwa/                 # shell: registry.ts (SOLE importer of sub-app manifests), router,
│                             #   profile picker, themed app grid, service worker, theme provider
├── packages/
│   ├── ui/                   # design tokens + 3 themes (admin / soccer / cute-animals), primitives, Ladle
│   ├── data/                 # PowerSync client, SQLite wasm/OPFS, useCollection hooks, typed API client
│   ├── platform/             # notifications/tts/storage/share interface; web impl now, capacitor later
│   ├── schema/               # Zod schemas: API contracts, rows, manifest schema — shared source of truth
│   ├── manifest/             # defineSubApp() + CONTRACT TEST KIT (auto-verifies every registered sub-app)
│   └── config/               # shared lint/ts/vitest presets
├── subapps/<id>/             # identical shape per sub-app:
│   │                         #   SPEC.md, AGENTS.md, manifest.ts, src/ (client),
│   │                         #   server/{routes.ts,jobs.ts,migrations/}, sync-rules.yaml, tests/
│   ├── todos/ email-triage/ news/ link-intake/ brain/ compute/
│   ├── math-trainer/ reading/ life-sandbox/ chess-kid/ chess-coach/ if-corner/
│   └── generated/<slug>/     # apps born from "add this link"
├── server/                   # Hono assembly, auth/, sync/ (write-replay enforcement), db/, jobs/,
│                             #   knowledge/ (item store, blobs, hybrid search, resolvers, paperpile-import/),
│                             #   compute/ (Modal sidecar client, run state machine, cost tracking),
│                             #   subapps/loader.ts, integrations/{gmail,gcal,bluebubbles,miniflux,x-browser,modal-sidecar}/,
│                             #   push/, journal/, audit/
├── agent/                    # SDK daemon, tools/, hooks/ (policy), quarantine/, approvals/, dream/,
│                             #   MEMORY.md, memory/, budget/
├── infra/                    # caddy/, powersync/ (sync-rules assembler), miniflux/, rss-bridge/,
│                             #   backup/ (+restore-drill.sh), mac-host/ (BlueBubbles runbook),
│                             #   provision/ (VPS hardening: ssh 2FA, ufw, fail2ban, unattended-upgrades, tailscale)
├── specs/                    # SPECS.md index (dispatch board), templates/ (SPEC, AGENTS, MINI-SPEC),
│                             #   platform/ (cross-cutting specs: sync, auth, shell, push, gmail, agent-runtime…)
├── docs/                     # ARCHITECTURE.md, DECISIONS/ (ADRs freeze settled choices), RUNBOOK.md, SECURITY.md
└── e2e/                      # Playwright: iphone/ipad/desktop projects, shell smokes, subapps/<id>.spec.ts
```

**Boundary rules (dependency-cruiser, CI hard-fail):** sub-app `src/` imports only `@everything/{ui,data,platform,schema,manifest}`; sub-app `server/` imports only `schema` + a small server kit (db, jobs, journal, auth ctx); no subapp↔subapp; `registry.ts` is the only manifest importer; all sub-app tables prefixed `subapp_<id>_`; migrations timestamped and never edited after merge. This is what makes parallel Opus sessions safe.

## Spec system (how less-capable agents build safely)

- **SPEC.md template** (in `specs/templates/`): id/status/profiles/depends-on/estimated-sessions header (status must be `ready` + open-questions empty before dispatch; >1 session → split first), then: Context & non-goals → numbered user stories → UX flows with **empty/loading/error/offline state for every screen** → exact data model (tables, soft-delete, sync scoping) → API surface (write ops with journal inverse-ops; reads via sync) → literal manifest entry → **numbered Given/When/Then acceptance criteria** → test plan (unit targets, e2e scenario names tagged with AC#s, 5-10-step human demo script) → out-of-scope → **expected file footprint** (CI flags diffs outside it).
- **AGENTS.md template** per sub-app: exact commands, boundary/forbidden lists, Definition of Done (typecheck/lint/depcruise/unit/contract/e2e + screenshots + AC→test map pasted in PR). Root AGENTS.md: global rules, branch conventions (`feat/<wp-id>-<slug>`), agents never merge, no secrets in output.
- **Contract kit** (`packages/manifest`): for every registered manifest, automatically asserts: schema-valid manifest, every route lazy-loads and renders per profile/theme without throwing, requiredCollections exist in assembled sync rules, settingsSchema renders, table prefixes correct, kid profiles never get admin collections. ~30 free checks per sub-app; agents can't "forget" integration.
- **"Add this link → app" pipeline**: link submitted → quarantined no-tool subagent fetches & summarizes (builder never fetches the raw link) → agent generates `MINI-SPEC.md` PR (tier `toy` = client-only default, or `synced` ≤2 tables — human opts in at PR review) → human approves → build session executes it into `subapps/generated/<slug>/` under identical contracts.

## UI architecture (the app is complicated — this keeps it navigable and flexible)

### Navigation model per form factor
- **Admin, iPhone**: bottom tab bar — **Today** (widget dashboard), **Apps** (grid of everything), **Add** (universal intake: paste/share target), **Search** (universal), **Chat** (agent). Approvals badge rides the Today tab. One-handed: primary actions bottom-anchored, destructive actions never in thumb-reach hot zones.
- **Kids (all devices)**: no tabs — a single themed grid of giant tiles, only their apps. PIN → grid → app. Simplicity is the feature.
- **iPad**: same admin tabs, but layouts go split-view (list + detail side by side) via responsive primitives — no separate iPad code in sub-apps.
- **Mac/desktop**: persistent left sidebar (Today, pinned apps, sections, approvals) + **Cmd+K command palette** (quick actions + universal search merged). Multi-pane layouts (e.g. brain: library list ∥ PDF reader ∥ notes).
- **Routing**: shell owns the router; sub-apps mount at `/a/<subapp-id>/*` and receive a scoped router. Deep links work from push notifications and search results. Back = real browser history (PWA-safe).

### Shell extension points (the flexibility mechanism)
Sub-apps contribute capabilities **declaratively via manifest** — the shell composes them; no shell edits per sub-app. Contract kit auto-tests each contribution.
- **`widgets`**: `{id, size: 'S'|'M'|'L', component: lazy, minProfile, refreshOnFocus}` → composed into Today. Each widget renders inside its own error boundary + suspense skeleton — a crashing widget shows a fallback card, never kills the dashboard. Examples: todos-due (M), next-event (S), running-CFD-job status (M), morning brief (L), compute budget (S).
- **`searchProviders`**: `async (query, ctx) => SearchResult[]` where `SearchResult = {title, snippet, icon, deepLink, score}`. Shell debounces, fans out, merges by normalized score, groups by app. Local SQLite providers answer instantly; the knowledge core's hybrid search joins as the heavyweight provider.
- **`intakeHandlers`**: `{accepts: ('url'|'pdf'|'epub'|'text')[], priority, classify(payload) => Offer}` — the Add tab / share-sheet shows matching offers ("Save paper to Brain", "Subscribe feed", "Make reading-list item", "Build an app from this"); link-intake orchestrates.
- **`quickActions`**: command-palette entries `{id, title, icon, shortcut?, run}` (e.g. "New todo", "Launch CFD run", "Capture note").
- **`notificationActions`**: per-category action buttons for push (e.g. approve/cancel email send from the lock screen).

### Layout & consistency
- **Primitives only**: sub-apps build screens from `@everything/ui` primitives — `Page`, `SplitPane` (auto-collapses to stacked navigation on phone), `Sheet`, `Card`, `ListRow`, `Toolbar`, `EmptyState`, `FormFromSchema` (renders a JSON schema → form; used by compute launch forms and settings). Container queries inside primitives handle form factors; sub-apps never write raw media queries or raw hex colors (lint-enforced).
- **Tokens**: color/type/space/radius/motion tokens; themes = admin (calm, dense-capable), soccer, cute-animals; dark mode per theme. Kid themes restyle the same primitives — no forked components.
- **Enforcement**: every screen ships a Ladle story; CI runs Playwright screenshot diffs per theme × viewport (iPhone/iPad/desktop); DESIGN.md states the principles agents must follow ("calm and glanceable; coach, don't nag; kid screens are playful but not noisy; empty states teach"). SPEC.md's UX section requires empty/loading/error/offline states per screen plus declared widget/search/intake contributions.

### Complexity management (12+ sub-apps without chrome bloat)
- **Pinned vs library**: admin pins favorites (sidebar/top of grid); everything else lives in the Apps grid, searchable. New/generated apps arrive unpinned.
- **Today dashboard edit mode**: add/remove/reorder widgets; per-profile defaults.
- **Per-profile visibility matrix** in settings controls which apps each profile sees (contract kit independently guarantees kids can't sync admin data even if shown).
- **Notification budget**: per-sub-app push quotas + quiet hours; kid profiles get no admin notifications; the agent prefers the morning brief + approvals inbox over interruptions — real-time push reserved for genuinely time-sensitive events.
- **Approvals inbox**: ONE queue for every approval-gated agent action (email sends, over-cap compute launches, agent PRs, mini-spec builds) — uniform card UI: what/why/diff-or-preview, approve/reject/edit, one tap. Surfaced as Today widget + badge; this is the primary human-in-the-loop surface.

### Admin apps
1. **todos** — native store; projects/goals hierarchy with % rollups; **goal-setting debrief flow** (agent-led interview producing named goals; revisited in weekly review); streaks & stats dashboards; agent task-breakdown ("first steps") on demand; notifications + backlog view; journaled edits. Tone everywhere: *executive coach with good vibes* — celebratory, never guilt-based.
2. **email-triage** — filtered important-inbox view (agent-ranked via quarantine summaries), archive/label/todo-ify swipes, email→todo with backlink, reply composer with agent drafts; **approve-to-send** + auto-send follow-ups per policy; never-delete.
3. **news** — unified feed (HN + Substack + X-feed + arbitrary RSS), thumbs up/down training a ranking signal, topic filters, offline cache of recent items, read-later.
4. **link-intake** — "add this" share-sheet/paste target; result: bookmark, feed subscription, reading-list item, or (via mini-spec pipeline) a new generated sub-app.
5. **chess-coach (adult, for Alex)** — goal: transfer 1900-puzzle skill to real games. Features: import own games (lichess export API / chess.com published-data API); server-side Stockfish analysis clustered into recurring leak themes; **thinking-framework trainers** (candidate-moves checklist drills, guess-the-move on annotated master games, "what's the plan?" positions); opening study by *ideas* (lichess chess-openings CC0 dataset + LLM-authored idea explainers for his actual repertoire), not lines; spaced repetition (ts-fsrs) of positions from his own games; time-management stats.
6. **brain (second brain / papers)** — UI over the knowledge core. Library views: papers list with Paperpile-style metadata columns, collections/folders (imported from Paperpile), tags, starred, reading queue. **Search-first interaction**: one search box, hybrid keyword+semantic, filters (type/tag/author/year); results also feed the shell's universal search. **PDF reading** via pdf.js in a reader pane (desktop: list+reader split view; phone: capture/search/skim focus) with highlight → saved as linked highlight item. Notes: markdown notes with `[[wikilinks]]` to any knowledge item; backlinks panel. Capture flows: share-sheet/link-intake ("add this" with a PDF/arXiv/DOI URL → resolved paper), email attachment → brain, BibTeX paste. Agent features (Phase 4): quarantined auto-summary note per imported paper, "related items" via embeddings, weekly "what you saved" digest. Export: BibTeX per collection (so citing from LaTeX still works).
7. **compute (jobs runner)** — admin-only UI over `server/compute/`. Job-type registry (first: sailing CFD from `modal_cfd`): each type = name + git ref + entrypoint + **params JSON schema → auto-generated launch form**. Run list with live status; run detail: streaming logs, params, cost (est + actual), artifact gallery (inline plots/images, downloadable data files); compare view for two runs' params+results. Monthly compute budget widget (spend vs cap). Notifications: run finished/failed → push. Agent integration: agent can launch within budget cap (auto-approved + logged), watches runs, posts result summaries to brain/todos; over-cap launches land in the approvals inbox.

### Kid apps (both profiles; themed skins from tokens)
8. **math-trainer** — start 3rd grade for both kids, headroom through ~7th grade; skill DAG keyed to Common Core codes; **Elo-based item selection** (Math Garden model: θ/β ratings, target P(success)≈0.75, speed-sensitive scoring) + **ts-fsrs (FSRS-6)** for fact review + Khan-style mastery ladder (promote via mixed-context quizzes, demote on failed review); procedural problem generators with error-pattern distractors (infinite content, zero licensing); deterministic cosmetic unlocks per theme; **LLM content-expansion pipeline**: agent authors new skill nodes/generators as PRs when a kid nears the frontier. No dark patterns: pausable streaks, no chance-based rewards, natural stopping points, sessions 15-20 min.
9. **reading** — sources: Gutendex (Gutenberg), Standard Ebooks curated shelf, StoryWeaver/Book Dash (CC), **sideloaded owned EPUBs** (share-sheet import → private server library), **LibriVox audiobooks**; reader on **foliate-js** (pinned commit); assistive modes: server-side TTS with word-timestamp read-along highlighting (Piper/Kokoro pre-generated at import), Kokoro-in-browser (WebGPU) option, autoscroll, focus-bolding (never call it "Bionic"), reading ruler, dyslexia-friendly font choices + spacing sliders; per-kid shelves and progress.
10. **life-sandbox** — Canvas2D + Uint8Array engine (512², trivially 60fps), pluggable rules: Conway, **Immigration (2-color), QuadLife (4-color), Generations (Brian's Brain, Star Wars), cyclic CA (rainbow spirals), Wa-Tor predator-prey**; finger painting (multi-touch), stamp library from classic RLE patterns, chaos/soup button, step-mode as the pedagogy, undo ring, saved creations shelf; fully client-side/offline.
11. **chess-kid** — chessground + chessops (GPL fine for private app), Stockfish 18 **lite single-threaded** WASM (~7MB, no COOP/COEP pain), optional Maia-1100 ONNX for human-like play + temperature sampling for beatable opponents; lichess-Learn-style progression (piece mazes → mini-games like pawn wars → check/mate-in-1 → two-rook ladder → tactics ladder per Steps Method sequencing); curated CC0 lichess puzzle subset (~10-20k easy puzzles, 2-4MB bundled); FSRS review of known patterns; ~15-25MB total, fully offline.
12. **if-corner** — bundle **MIT-licensed Zork I-III** (historicalsource repos, Nov 2025 grant; keep LICENSE, don't brand as "Zork") via Parchment; kid-friendly onramps (Lost Pig etc., license-checked); **LLM-authored original adventures**: agent writes deterministic JSON world-specs (rooms/items/flags/win conditions) run by a small engine — authored at build time, human-reviewable, coherent and safe (no live LLM in the kids' loop), fully offline; later: Myst-like scene-graph + pre-generated images (~$10/world).

## Phased roadmap

Sizing: one work package ≈ one Opus 4.8 session. Critical path bolded. Client-only sub-apps are the parallel pressure-release valve.

- **Phase 0 — Foundations** → demo: *installed themed shell, works in airplane mode.*
  **0.1 scaffold+CI** → 0.2 design system ∥ **0.3 server skeleton** → **0.4 auth+profiles** ∥ **0.5 PowerSync spike (GO/NO-GO, timeboxed 2 sessions; fallback delta-sync spec pre-written)** → **0.6 PWA shell + contract kit** ∥ 0.7 deploy pipeline (VPS provision, Tailscale, Caddy, WAL backup + restore drill) ∥ 0.8 Playwright harness.
- **Phase 1 — First demo slice** → demo: *Alex's offline todos sync phone↔Mac; kids paint Wa-Tor on iPad.*
  **1.1 todos core** (first full vertical slice — deliberately serial; calibrates all future sub-app builds) → 1.4 gamification/goals v1; ∥ 1.2 life engine → 1.3 life UI; ∥ 1.5 settings framework.
- **Phase 2 — Integrations backbone** → demo: *morning triage on phone; email→todo; push arrives; filtered news over coffee.*
  **2.1 Gmail ingestion** → **2.2 email-triage app**; ∥ 2.3 web-push e2e; ∥ 2.4 calendar sync + today view; ∥ 2.5 news fetch layer (Miniflux/RSS-Bridge/HN/Substack + **X logged-in-browser scraper container**) → 2.6 news reader; ∥ **2.7 knowledge core** (item/blob store, FTS+pgvector hybrid search, DOI/arXiv resolvers) → **2.8 Paperpile import job** (dry-run → real run on Alex's export) → 2.9 brain app v1 (library, collections, search, pdf.js reader, capture) → 2.10 notes+backlinks; ∥ 2.11 Modal sidecar + compute runs store → 2.12 compute app v1 (job-type registry with CFD, launch form from params schema, run detail with logs+artifacts, cost tracking).
- **Phase 3 — Learning apps** (three parallel tracks) → demo: *math session adapts; book reads aloud with word highlight; chess lesson 1; Alex's games imported and leak-analyzed.*
  Math: 3.1 engine → 3.2 UI → 3.3 parent dashboard. Reading: 3.4 sources+library (incl. sideload) → 3.5 reader → 3.6 TTS/read-along + LibriVox. Chess: 3.7 board+engine (shared by both chess apps) → 3.8 kid progression → 3.9 Maia (cuttable) → **3.11 chess-coach v1** (game import + analysis + first trainers). ∥ 3.10 if-corner v1 (Parchment + Zork).
- **Phase 4 — Agent runtime** → demo: *overnight dream produces morning brief + memory PR; a submitted link becomes a working toy app via approved PRs; audit log shows zero unapproved external actions.*
  **4.1 daemon + safety core FIRST** (srt sandbox, egress allowlist, PreToolUse policy, audit log, budget enforcement — nothing else in phase 4 starts before this merges) → 4.2 quarantine pipeline (+CI injection fixtures) → 4.3 memory+dream → **4.4 email agent actions (approval+delay queue, follow-up policy)** ∥ 4.5 news ranking ∥ **4.6 link-intake mini-spec pipeline** → 4.7 first generated app (fix the pipeline, not the app) ∥ 4.8 BlueBubbles bridge + texts in unified inbox ∥ 4.9 LLM-authored IF ∥ 4.10 brain agent features (quarantined paper auto-summaries, embedding-based related-items, saved digest) ∥ 4.11 compute agent autonomy (budget-capped launch policy hook, run-watching, result summaries → brain/todos).
- **Phase 5 — v1 close** (parallel, cuttable): 5.1 Capacitor wrap + TestFlight; 5.2 perf/polish; 5.3 Myst-like; 5.4 security review + restore drill + runbooks; 5.5 spec/reality reconciliation pass.

## Verification

CI gates, cheapest first: typecheck/lint/**dependency-cruiser (boundaries = hard fail)**/migration lint → vitest unit (engines, sync queue, policy hooks) → **contract kit over every registered manifest** → migrations applied to ephemeral Postgres + schema-valid fixture hits on declared endpoints → Playwright e2e (smoke per PR: shell install/PIN/offline + touched sub-app; full matrix nightly incl. **sync-roundtrip spec** — the tripwire for the scariest subsystem) → agent proof-of-work in PR (AC→test map, DoD output, screenshots, footprint diff). Agents never merge. Each milestone additionally requires Alex to run a literal phone-in-hand demo script (kept in ROADMAP.md). Agent-safety CI: prompt-injection fixture emails must fail to influence the privileged agent.

## Top risks & mitigations

1. **PowerSync self-hosted maturity** → timeboxed spike gate + pre-written delta-sync fallback; server is enforcement point either way.
2. **X scraper → account flag/lockout** → gentle jittered polling of own feed only, real browser profile, instant kill-switch, Nitter-RSS fallback adapter; treat X as best-effort.
3. **iMessage/BlueBubbles breaks on macOS updates** → optional adapter, nothing depends on it; health-check + alert; defer Mac auto-updates; runbook.
4. **Prompt injection via email/news** → quarantine subagent + deterministic gates + egress allowlist + draft-only tools + audit; CI red-team fixtures.
5. **iOS storage eviction** → server is source of truth; cold-resync e2e-tested; installed-PWA usage; Capacitor escape hatch.
6. **Spec drift / boundary erosion across parallel agent sessions** → status+footprint discipline, depcruise hard-fail, registry as single integration point, Phase-5 reconciliation.
7. **Gmail unverified-app token friction** → production+unverified mode, refresh-failure alerting, re-auth runbook; degrade to read-only cache.
8. **Infra sprawl for one maintainer** → one compose file, provision/restore scripts, monthly automated restore drill, uptime monitoring.
9. **WP overruns stalling critical path** → 1-session sizing cap, force spec-split, parallel client-only backlog.
10. **Kid PIN mistaken for security** → documented: tailnet + device session is the boundary; contract kit asserts kid profiles never receive admin collections.
11. **Modal cost runaway from agent-launched jobs** → hard monthly compute cap enforced in the deterministic policy hook (not model judgment), cost recorded per run from Modal's usage data, over-cap → approvals inbox, global kill switch; cap + spend visible as a dashboard widget.
12. **Paperpile import fidelity** (folder structure, PDF filenames, dedup) → importer is idempotent + journaled with a dry-run diff report Alex reviews before the real run; originals never modified; failed items land in a review queue rather than being dropped.

## Execution notes for the build sessions

- First action: `add_repo` `agvaughan/everything`, clone, and initialize with Phase 0 WP-0.1 (scaffold + this plan's templates + AGENTS.md + ROADMAP.md + SPECS.md). Commit this plan as `docs/PLAN.md`.
- Orchestrator (Fable or Opus) dispatches WPs per ROADMAP; sub-app builds run as isolated agent sessions against their SPEC.md; PRs reviewed before merge (Alex or orchestrator-with-Alex's-standing-approval for green-CI, in-footprint changes).
- Secrets (Google OAuth, Anthropic key, VAPID, Tailscale, Modal token) live in `.env` on the server only; `.env.example` documents them; agents never echo values.
- Human setup tasks Alex must do once (RUNBOOK will walk through): create GCP OAuth client + consent screen (production/unverified), Tailscale accounts/devices, VPS or Mac mini provisioning, BlueBubbles Mac setup, X login into the scraper profile, Apple Developer enrollment (Phase 5), Anthropic API key, Modal API token, **export the Paperpile library** (BibTeX + PDFs) and drop it on the server for the import job.
- When compute work needs the CFD code's actual entrypoints/params, `add_repo` `agvaughan/modal_cfd` to that session (also Alex's repo — authorized).
