# ROADMAP — Cloud‑First Franchise Brain (with Event Arcs, Budget Guardian, Fallback Episodes)

**Goal:** Build a fully automated, cloud‑first studio that produces interactive D&D Shorts with a BG3‑inspired (but original) companion app, plus midform and longform recaps — optimized for organic growth, safety, and hard monthly budget caps. Includes Event Arc engine, Budget Guardian Agent, and graceful Fallback Episode Generator.

> Use this roadmap with Cursor + Claude. After each step, the app is functional. Deploy from Day 0. Heavy compute happens in the cloud.

---

## 0) Architecture Snapshot

- **Monorepo**
  - `/frontend` — Next.js (Viewer Pane + Admin Pane)
  - `/backend` — FastAPI (or Express/Nest) — public API + admin API + orchestrator hooks
  - `/worker` — Job runner / queue consumers (Python Celery or Node BullMQ)
  - `/renderer` — Cloud render jobs (FFmpeg orchestrations, TTS harness, image fetch)
  - `/packages/shared` — Types, JSON Schemas, prompts, constants
  - `/infra` — Dockerfiles, deploy manifests (Railway/Render/Fly), GitHub Actions
- **Cloud primitives**
  - DB: Postgres (Railway/Supabase). Dev uses SQLite.
  - Storage: Cloudflare R2 or Supabase Buckets (video, thumbs, sprites, sfx)
  - Render/AI: Replicate/RunPod/Stability API, TTS (ElevenLabs/Coqui/Piper)
  - Deploy: Railway or Render (auto‑deploy on Git push)
- **Content ladder**
  - Shorts (45–59s) → discovery
  - Midform Recaps (5–12m every 7–10 eps) → retention
  - Season/Arc Longform (30–60m) → franchise anchor
  - Podcast feed (audio‑only recaps) → extra reach
  - Lore Zines (auto PDF per arc) → funnel + collectibles

---

## 1) Global Contracts & Schemas

### 1.1 Script JSON (v3)
```json
{
  "episode": 23,
  "kind": "short" | "recap_mid" | "recap_season" | "fallback",
  "hook": "string",
  "recap": "string",
  "beats": [
    {"t":0.0,"dur":2.0,"shot":"close_rogue","bg":"loc:roost","fg":["char:rogue"],"vo":"...","sfx":["whoosh"],"ost":"bed_stealth","gfx":"d20_roll(+4)","caption":"..."}
  ],
  "captions": ["..."],
  "music": "bed_stealth",
  "dice_receipt": {"needed":16, "ability":"Stealth", "mods":4, "rolled":13, "result":"fail", "seed":"ep23:B:2025-09-04"},
  "cta": [
    {"code":"A","label":"Flee","ability":"Acrobatics","dc":14,"situational_mod":0,"consequence":"We escape but drop loot."},
    {"code":"B","label":"Cast Sleep","ability":"Spell","dc":15,"situational_mod":-1,"consequence":"Risk backfire."}
  ],
  "style": {"palette":"amber-wood-parchment","ui_variant":"bg3-esque-original"}
}
```

### 1.2 Event Arc Spec
```json
{
  "code": "harvest_moon_2025",
  "window": {"start": "2025-10-15", "end": "2025-11-05"},
  "theme": "spooky harvest, lunar rites, mischievous spirits",
  "cosmetics": {"palette":"moonlit","particles":["fireflies","mist"],"music_tags":["lyre","bells"]},
  "mechanics": {"seasonal_items":["Moonseed"],"buffs":["Night's Boon:+1 Stealth at night"]},
  "cadence_overrides": {"shorts_per_week": 5, "recap_every": 8},
  "safe_terms": ["pumpkin","lantern"],
  "banned_terms": ["brand_names","real_politics"],
  "priority": true
}
```

### 1.3 Budget Config (YAML)
```yaml
monthly_budget_cents: 15000   # $150
reserve_percent: 20           # keep 20% untouched
soft_stop_percent: 95         # warn/halt before 100%
per_unit_costs:
  llm_per_1k_tokens_cents:
    cheap: 5
    mid: 15
    premium: 30
  tts_per_min_cents:
    cheap: 2
    premium: 10
  img_per_gen_cents:
    cheap: 2
    premium: 8
  render_cpu_minute_cents: 0.5
  storage_per_gb_month_cents: 2
plans:
  short_full:
    llm_tokens_k: 8
    tts_min: 0.9
    img_gens: 6
    render_cpu_min: 8
    ab_variants: 3
  short_saver:
    llm_tokens_k: 4
    tts_min: 0.7
    img_gens: 2
    render_cpu_min: 6
    ab_variants: 2
  short_minimal:
    llm_tokens_k: 2
    tts_min: 0.5
    img_gens: 0
    render_cpu_min: 4
    ab_variants: 1
  fallback_dm:
    llm_tokens_k: 0.5
    tts_min: 0.6
    img_gens: 0
    render_cpu_min: 3
    ab_variants: 0
policy:
  max_fallback_per_7eps: 1
  min_days_between_fallback: 5
  event_arc_priority_weight: 1.25
  cadence:
    default_shorts_per_week: 5
    min_shorts_per_week: 3
    recap_every_eps: 8
```

### 1.4 Ledger Tables (DB)
- `budget_snapshot(month, spent_cents, forecast_cents, updated_at)`
- `budget_ledger(id, episode_id, kind, component, qty, unit_cost_cents, total_cents, created_at)`
- `episode_cost_plan(episode_id, plan_code, estimate_cents, decided_by, rationale)`
- `incident(id, severity, scope, message, stack, status, proposed_patch, created_at, resolved_at)`
- `event_arc(code, window_start, window_end, spec_json, priority)`

---

## 2) Budget Guardian Agent — Design

**Purpose:** Keep monthly spend under cap; choose the cheapest viable plan per episode; handle event priorities; avoid fallback loops.

### Inputs
- `budget_config.yml` (above)
- Live `budget_snapshot` + `budget_ledger`
- Episode attributes: `is_event_arc`, `deadline_ts`, `kind` (short/recap), `importance`
- Queue state: episodes remaining in the month, cadence targets

### Algorithm (pseudocode)
```python
remaining = monthly_budget - spent
reserve = monthly_budget * reserve_percent
usable = max(0, remaining - reserve)

# Predict minimum required episodes to hit cadence
eps_left = episodes_left_in_month()
min_weekly = config.policy.cadence.min_shorts_per_week
min_eps_needed = min_weekly * weeks_left_in_month()

# Cost models
plans = [short_full, short_saver, short_minimal, fallback_dm]
if episode.is_event_arc:
    boost = config.policy.event_arc_priority_weight
else:
    boost = 1.0

for plan in plans:
    est = estimate_cost(plan)
    score = quality_score(plan) * boost
    if est <= usable:
        choose highest score among feasible plans

if no plan fits usable:
    if can_reallocate_from_future():
        pull_forward_budget()
        choose best feasible plan
    else:
        choose fallback_dm if fallback_policy_allows()
        else delay_episode_or_reduce_cadence()

record decision: plan, estimate, rationale
```

### Degradation Ladder
1. `short_full` → 2. `short_saver` → 3. `short_minimal` → 4. `fallback_dm`

### Anti‑Loop & Safeguards
- **Max 1 fallback per rolling 7 episodes** (`policy.max_fallback_per_7eps`)
- **Min 5 days between fallbacks** (`policy.min_days_between_fallback`)
- **Fallback cannot be scheduled twice in a row**
- **If forced fallback but quota exceeded** → auto‑**reduce cadence** for current week (e.g., from 5 → 3) and notify admin
- **Hard stop** at `soft_stop_percent` of monthly budget unless admin overrides

### Outputs
- `episode_cost_plan` row with `plan_code`, `estimate_cents`, `decided_by='budget_guardian'`, `rationale`
- Event in `event_log` with human‑readable explanation

---

## 3) Fallback Episode Generator — Design

**Goal:** Maintain cadence when upstream fails or budget forbids normal render; keep immersion; cost < $0.02–$0.05.

### Visual & Audio
- Static parchment background + subtle animated vignette
- One character portrait (cached asset) slowly pans (Ken Burns)
- Dice overlay animation from library (no new renders)
- TTS: one narrator voice (cached/local model)
- Music: single loop from free library, -14 LUFS target; or **no music** if saver mode

### Structure (45–50s)
1. **Hook (5s):** “Tonight’s roll: {ability} DC {dc}. Here’s what’s at stake…”
2. **Recap (10–15s):** last episode summary
3. **Roll Moment (8–10s):** show dice overlay, announce roll & modifiers
4. **Outcome Beat (10–12s):** describe result; tease consequence
5. **CTA (8–10s):** 2–4 options with required DCs shown

### State Rules
- Fallback **does not advance** major plot beyond the voted choice reveal; it may **set up** consequences, but defers heavy animation to next normal episode
- It **does** collect votes and logs dice receipts for transparency

### Trigger Conditions
- Budget Guardian selects `fallback_dm`
- Orchestrator detects render failure after N retries (e.g., 3) or API outage

### Anti‑Repetition
- Rotate 6 prewritten narrator templates and 3 background variants
- Insert one micro‑gag from a small bank to avoid sameness

### Logging
- `episode.status='fallback_published'`
- `episode.fallback_reason=('budget'|'outage'|'render_failed')`

---

## 4) Event Arc Engine — Design

**Purpose:** Time‑boxed themed runs that change cosmetics, mechanics, and cadence, improving novelty and growth.

### Planner Agent
- Scans calendar + safe Trend Nod Packs to propose arc windows
- Ensures **timeless** themes (no real‑world politics/brands)

### Effects
- Theming (palette, overlays, music tags)
- Limited items/buffs available only in the window
- Cadence bump (e.g., 5/wk) **if budget allows** per Budget Guardian

### A/B & Metrics
- Compare event vs baseline on CTR, VTR@3s/10s/30s, votes/episode
- Auto‑generate an **Event Trailer Short** at arc start
- Auto‑compile **Event Recap** at arc end

---

## 5) Growth & Content Ladder — Execution Rules

- **Shorts:** 45–59s enforced. If beats <45s, auto add "reaction gag" or extend VO; if >59s, compress captions/VO.
- **Midform Recap:** Every `recap_every_eps` (default 8). Budget Guardian must approve; otherwise postpone.
- **Season/Arc Longform:** Triggered after flagged milestones; may be postponed to next month if over budget.
- **Podcast Feed:** Strip VO + bed → mp3; publish via RSS.
- **Lore Zine PDF:** Autogen per arc (script highlights, NPCs, items, maps) — stored in R2.

---

## 6) Metadata & Trend Optimizer

- Pulls safe trending nouns/verbs from Google Trends & Reddit titles (RSS)
- Builds **Nod Packs** with `safe_elements` / `banned_elements`
- Injects **at most one** nod per episode (title + one beat), never central to plot
- Titles ≤60 chars; include mechanic (e.g., “Beat DC 16… or Else”) — no clickbait

---

## 7) Reliability & Self‑Healing

- **Backoff & Dead‑letter:** exponential retries, DLQ with alert
- **Bug Triage Agent:** classifies; proposes patch diff for admin approval
- **Regression suite:** generate Script JSON, 10s render stub, mock publish
- **Health dashboard:** per‑stage green/yellow/red; budget bar with forecast

---

## 8) Viewer & Admin UX (BG3‑esque, original)

- **Viewer**
  - Party panel (HP bars, buffs/debuffs), inventory grid, bosses (HP rings)
  - Quest log & map
  - Dice log with seed, roll, modifiers, tie‑break receipts
  - Vote hub with 2–4 options + DC + “need X+ to pass”
- **Admin**
  - Pipeline board: ingest → resolve → script → render → publish → sync
  - Budget Guardian panel: plan chosen, estimate, rationale, override button
  - Event Arc planner: calendar with theme switcher
  - Incidents: view stack, generated patch diff, approve/apply

---

## 9) Exact Prompts (drop in `/packages/shared/prompts/`)

### 9.1 Writer — Low‑Cost, Deterministic
```
System: You write 45–59s, family‑friendly fantasy shorts with clear tabletop mechanics. Keep recurring character voices consistent. Avoid brands/politics. Vary at least 2 of: hook style, location, camera move, gag structure, music bed.

User:
Arc: {arc_summary}
State: {state_brief}
EventArc: {event_arc_spec_optional}
Roll: {receipt_short}  # e.g., Needed 16, rolled 13 (+2) = FAIL
NodPack(optional): use at most one of {nod.safe_elements}; never include {nod.banned_elements}.
Constraints:
- End with 2–4 CTA options; each includes ability, DC, situational_mod, consequence
- Must output strict Script JSON (v3 schema)
Return ONLY JSON.
```

### 9.2 Metadata Agent
```
System: Optimize YouTube metadata for Shorts. Titles ≤60 chars, include mechanic (e.g., "DC 16"). Avoid brands/politics.

User:
Hook: {hook}
Mechanic: {ability} DC {dc}, roll {d20} (+{mods}) -> {result}
Theme tags: {tags}
NodPack(optional): {safe_elements}
Output JSON: { "titles": [..3], "descriptions": [..3], "hashtags": ["#dnd","#fantasy","#shorts", ...] }
```

### 9.3 Budget Guardian — Explainer (LLM‑assisted rationale)
```
System: You are a budget planner for an AI video studio. Given costs, remaining budget, cadence goals, and event flags, choose the cheapest viable plan and explain why in 2–3 sentences.

User:
Config: {budget_config_snippet}
Spent: {spent_cents} of {monthly_budget_cents}
Episodes left this month: {eps_left}
This episode: {attrs_json}
Plans w/ estimates: {plan_estimates}
Policy: {policy_snippet}
Output JSON: { "plan": "short_saver", "estimate_cents": 123, "rationale": "..." }
```

### 9.4 Fallback Narrator — Template Picker
```
System: Write a 45–50s fallback narration for a parchment card visual. Keep immersion; do not advance major plot beyond the revealed roll outcome. Include CTA with 2–4 options and required DCs.

User:
Last Recap: {recap}
Roll Receipt: {receipt_short}
Event Flavor(optional): {event_arc_flavor}
Output: Script JSON (v3) with kind="fallback".
```

---

## 10) Step‑by‑Step Build Plan (Claude‑Executable)

Each step is shippable. After finishing, deploy to cloud, verify, then proceed.

### Step 0 — Cloud Bootstrap & Spine
- Monorepo scaffold, env loader, logging
- SQLite dev + Postgres prod
- Health endpoints; Dockerfiles; Railway/Render deploy
- **Deliverable:** Live blank site + `/api/health` + DB migrations

### Step 1 — Core Game Loop (Votes → Dice → Script JSON → Placeholder Video)
- Models: `choice`, `vote`, `roll`, `episode`
- Deterministic seed: `ep{num}:{winner}:{cutoff_date}`
- Templated Script JSON (no LLM)
- Placeholder FFmpeg render (cards + captions), clamp 45–59s
- **Deliverable:** Publish a placeholder short end‑to‑end

### Step 2 — Asset Registry & Seeds
- Models: `asset`, `character`, `location`, `inventory_item`
- Deterministic seeds per entity (xxhash)
- Admin CRUD + viewer state API
- **Deliverable:** Party shown on dashboard with stable seeds

### Step 3 — Cloud Renderer (TTS + Art + Composite)
- Integrate TTS (Coqui/ElevenLabs) + image gen provider or stock sprites
- FFmpeg compositor with layers, captions, LUFS normalization
- Thumbnails (3 variants)
- **Deliverable:** Automatically rendered 9:16 short with VO & music

### Step 4 — Publishing & A/B
- YouTube upload, pin vote comment, SRD attribution
- Metadata Agent (3 titles, 3 desc, hashtags)
- A/B rotate (≥10k impressions or ≥2h)
- **Deliverable:** Video live on YT; variants rotating; dice receipt linked

### Step 5 — Companion App (BG3‑esque)
- Pages: Dashboard, Episodes, Vote Hub, Dice Log, Lore
- Shows DCs, modifiers, and needed roll; tie‑break receipts
- **Deliverable:** Fully functional viewer experience

### Step 6 — Budget Guardian (Hard Cap) + Low‑Cost Modes
- Implement config + estimator + plan chooser + ledger writes
- Enforce degradation ladder + anti‑loop rules
- **Deliverable:** Episodes adapt to budget; guardrails visible in admin

### Step 7 — Fallback Episode Generator
- Templates, narrator prompt, cached assets
- Trigger on outage/budget; logs reason; respects quotas
- **Deliverable:** Cadence never breaks; no repeat spam

### Step 8 — Event Arc Engine
- Event spec model + planner; theme application hooks (UI, music, overlays)
- Event trailer auto‑short; end recap auto‑compile
- **Deliverable:** Schedule and run an event window end‑to‑end

### Step 9 — Trend Miner & Metadata Optimizer
- Build Nod Packs from safe trend sources; inject sparingly
- SEO dashboards; forbidden terms filter
- **Deliverable:** Titles/desc subtly trend‑aware, policy‑safe

### Step 10 — Midform Recap Compiler
- Stitch last N shorts; add narration bridge; updated captions
- Budget‑gated; publish to longform feed
- **Deliverable:** Auto 5–12m recaps

### Step 11 — Season/Arc Longform & Podcast Feed
- 30–60m compilations; re‑render pacing; audio export + RSS feed
- **Deliverable:** Longform on YT; podcast on Spotify/Apple

### Step 12 — Reliability & Self‑Healing
- Triage agent; patch suggestions; regression suite; DLQ
- Admin incident console with approve/apply
- **Deliverable:** Pipeline survives outages; safe patch workflow

---

## 11) API Surface (selected)

**Public**
- `GET /api/campaigns/:id/state`
- `GET /api/campaigns/:id/episodes?limit=20`
- `POST /api/campaigns/:id/vote` (rate‑limited, bot checks)
- `GET /api/campaigns/:id/dice-log`

**Admin**
- `POST /admin/episodes/:id/close-vote`
- `POST /admin/episodes/:id/resolve`
- `POST /admin/episodes/:id/generate`
- `POST /admin/episodes/:id/render`
- `POST /admin/episodes/:id/publish`
- `POST /admin/budget/plan/:episodeId` (returns chosen plan + estimate)
- `POST /admin/fallback/trigger/:episodeId` (manual)
- `POST /admin/event-arcs` (CRUD)
- `GET /admin/incidents` / `POST /admin/incidents/:id/apply`

---

## 12) Acceptance Criteria (Platinum MVP)

- **Cadence:** 3–5 shorts/week sustained for a month
- **Length compliance:** All shorts 45–59s
- **Transparency:** Dice receipts visible in episode + app
- **Budget:** Monthly spend ≤ cap; fallback ≤ 1 per 7 eps; zero repeat spam
- **Growth:** CTR ≥ 3%, VTR@3s ≥ 60% median, vote participation ≥ 0.3%
- **Resilience:** Auto fallback on outages; DLQ drain; no multi‑day gaps
- **Event:** One event arc executed with trailer + recap

---

## 13) Claude Run Notes (How to Execute)

- Put this file at repo root as `ROADMAP.md`.
- In Cursor chat: 
  1) “Open `ROADMAP.md`. Start with **Step 0 — Cloud Bootstrap & Spine**. Implement end‑to‑end. Deploy to Railway/Render. Stop when live with health endpoints.”
  2) Run locally to verify (`npm run dev`), then confirm cloud URL works.
  3) “Proceed to **Step 1**.” Repeat until Step 12.
- Always ask Claude to:
  - commit code with clear messages
  - output run commands + env list
  - provide mock keys and `.env.sample`

---

## 14) Env & Providers (suggested)

- `DATABASE_URL` (Railway/Supabase Postgres)
- `STORAGE_R2_*` (account, key, bucket)
- `YOUTUBE_API_KEY`, `YOUTUBE_CLIENT_SECRET` (OAuth)
- `TTS_PROVIDER_KEY`, `IMG_PROVIDER_KEY`
- `BUDGET_CONFIG_PATH` (mounted `config/budget.yml`)

---

## 15) Testing Checklist (per Step)

- Unit: seed determinism, dice math, cost estimator math
- Integration: end‑to‑end short render under each plan
- Budget sims: random month usage, ensure no policy breaks
- Resilience: kill render pod; ensure fallback triggers once, not loops
- Event arc: theme applied only inside window

---

## 16) Visual Style Rules (BG3‑esque, original)

- Materials: parchment, carved wood frames, gem buttons; **no BG3 fonts/icons**
- Typography: chunky serif headers + readable sans captions
- Colorways: warm amber + deep forest; moonlit blues for events
- Motion: slow pans, parallax, subtle particles; big readable dice overlay

---

## 17) Roll Receipt Overlay (spec)

- Shows: `Needed DC`, `Roll`, `Modifiers`, `Total`, `Result`
- Tie‑break: show 1–10=A, 11–20=B animation when used
- Duration: ~0.5–1.0s in the roll moment

---

## 18) Anti‑Stale Rules

- Vary ≥2 dimensions per episode: hook, location, camera move, gag structure, music bed
- Enforce rotation lists; cooldown on reused gags
- Nod Packs: at most one per ep; expire after 10 days

---

## 19) Security & Safety

- Age‑safe language, profanity/brand blacklists
- Signed webhooks, rate limits, bot checks on votes
- Secrets in provider vaults (Railway/Render)

---

## 20) What “Done” Looks Like

- You press **Schedule** → Budget Guardian selects a plan → episode renders in cloud → uploads with A/B → companion app updates → ledger shows spend & forecast → if outage/budget shortfall, you get a graceful fallback once, then normal cadence resumes — all without touching your old PC beyond clicking **Approve**.


