# Planning — Mail Job Tracker

## 1. Overview

An app that connects to Gmail (extensible to other providers later), reads job-related
email, and turns it into two structured views for the user:

1. **Application Tracker** — a kanban of real applications (Applied → Screening →
   Interview → Offer/Rejected), built from status-update emails.
2. **Recommendation Feed** — a ranked list of job-board recommendation emails
   (Indeed, Xing, StepStone, Naukri, etc.) scored against the user's own skill/
   preference profile, instead of relying on keyword matching or the portal's own
   "match" label.

All extraction and matching runs through small/local-friendly models where possible,
orchestrated with LangChain, so the pipeline is swappable (model, prompt, provider)
without touching the app logic around it.

## 2. Features & current status

| # | Feature | Status | Notes |
|---|---|---|---|
| 1 | Extractor & Classification (is this mail job-related? which category?) | **Deferred** | User already has Gmail filters/labels (`Jobs-portal/*`) doing source-based classification manually. Revisit once Feature 2 is stable — automate what the filters currently do by hand. |
| 2 | Recommendation Matcher | **Active — build first** | Reads `Jobs-portal/*` labeled mail, extracts listings, scores against user profile. |
| 3 | Application Tracker | **Next** | Schema and stage taxonomy already designed (see `system_design.md`). Builds on the same extraction infra once Feature 2 proves the pipeline out. |

## 3. Goals

- Replace "scroll through 10 job-board labels" with one ranked feed of listings that
  actually fit the user.
- Never rely on keyword matching alone for fit — always reason over structured
  profile vs. structured listing.
- Keep extraction and judgment as separate pipeline stages so either can be swapped
  independently (different model, different prompt, different provider).
- Start on 2–3 portals, prove the pipeline, then expand to all labeled sources.

## 4. Non-goals (for now)

- Auto-applying to jobs.
- Multi-provider mail support (Outlook, IMAP generic) — Gmail only for MVP.
- Multi-user / hosted SaaS — single-user, self-hosted first.
- Feature 1 automation — deferred, Gmail filters cover this manually for now.

## 5. Tech stack decision

Two options were on the table:

| | Hono.js + Bun | Python + FastAPI |
|---|---|---|
| API layer | Very fast, lightweight | Fast enough, mature ecosystem |
| LangChain support | `langchainjs` exists but smaller ecosystem, fewer integrations kept current | First-class, most LangChain development happens here first |
| Structured/constrained output tooling | Fewer mature options | `pydantic` + LangChain structured output parsers, `outlines`, `instructor` all mature |
| Background jobs / schedulers (Gmail sync) | Workable, more DIY | Mature options (APScheduler, Celery, arq) |
| Postgres tooling | Drizzle ORM, decent | SQLAlchemy / SQLModel, `asyncpg`, very mature |

**Decision: Python + FastAPI + LangChain for the whole backend.**

Reasoning: the core value of this app is the AI pipeline (extraction + matching), not
raw API throughput — this is a single-user app processing a few hundred emails, not a
high-QPS service. LangChain's Python ecosystem is significantly ahead of its JS
counterpart for structured output parsing, provider swapping, and chain composition,
and using one language for the whole backend avoids a two-service split for an MVP
this size. Hono/Bun stays a good option later if a very thin public-facing API or
edge-deployed BFF is ever needed in front of this service — not needed now.

## 6. Stack summary

- **Backend:** Python, FastAPI
- **Orchestration:** LangChain (structured output chains, prompt templates, provider
  abstraction)
- **DB:** PostgreSQL
- **Mail source:** Gmail API (OAuth2, incremental sync via `history.list`, scoped to
  `Jobs-portal/*` label tree for Feature 2)
- **Frontend:** to be decided when we get to UI — likely a simple SPA reading from
  the FastAPI REST layer (kanban view for Feature 3, ranked feed for Feature 2)

## 7. Phased roadmap

### Phase 0 — Setup
- Repo scaffold (FastAPI app, Postgres, Alembic migrations)
- Gmail OAuth flow, token storage
- Base DB schema (see `system_design.md`)

### Phase 1 — Feature 2 MVP (Recommendation Matcher)
- Scope: 2–3 portals only — Indeed (single-listing, clean), Xing (digest format),
  StepStone.de (highest volume: 1,748 emails)
- Incremental sync limited to `Jobs-portal/indeed`, `Jobs-portal/xing`,
  `Jobs-portal/stepstone.de`
- Per-portal listing extraction chain (LangChain structured output)
- User profile store (manual entry first, editable)
- Match chain: batched per email, array in → array of verdicts out
- Dedup across portals (fuzzy match on company + title + location)
- Ranked feed endpoint + minimal UI

### Phase 2 — Expand & feedback loop
- Add remaining portals: foundit.in, Indeed-apply, justjoin.it, Naukri, naukrigulf,
  pracuj.pl, rocketjobs.com, Uplers
- Thumbs up/down feedback on matches → stored, used to sanity-check
  `matched_criteria` / `mismatched_criteria` reasoning quality over time
- One-time backfill job for historical mail in each label (rate-limited)

### Phase 3 — Feature 3 (Application Tracker)
- Status-update extraction chain (separate from recommendation extraction)
- Stage transition logic (forward-only except into rejected/withdrawn)
- Application matching (merge emails into one record per application)
- Kanban UI

### Phase 4 — Feature 1 (Extractor & Classification), automated
- Replace manual Gmail filters with the model-based classifier originally scoped
- Only worth doing once Phases 1–3 are stable and the manual filter approach starts
  feeling limiting

## 8. Open decisions

- Frontend framework — not yet chosen, revisit at start of Phase 1's UI work.
- Which reasoning-capable model to actually run for the match chain (Gemma 4 E4B,
  Phi-4-mini-reasoning, or a hosted API model) — depends on whether this stays fully
  on-device or accepts a cloud call for the judgment step. Extraction step should stay
  small/local regardless.
- Whether `promoted_to_application_id` is needed on match records (i.e., does a
  strongly-matched recommendation ever get promoted into a tracked application on
  "apply") — affects the Phase 3 data model, decide before building Phase 3.

## 9. Risks

- Portal email templates change without notice → extraction chain needs to fail
  gracefully (low confidence, not a crash) and ideally flag template drift rather than
  silently mis-extract.
- Digest-format emails (Xing, StepStone) bundle many listings per email — extraction
  must always return an array, never assume one listing per email.
- Volume skew across portals (StepStone: 1,748 vs justjoin.it: 4) — backfill needs
  rate limiting so it doesn't burn through model calls in one run.
