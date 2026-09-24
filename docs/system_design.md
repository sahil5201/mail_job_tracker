# System Design — Mail Job Tracker

## 1. Architecture overview

```
                         ┌─────────────────────┐
                         │   Gmail API (OAuth)  │
                         └──────────┬───────────┘
                                    │ incremental sync (history.list)
                                    ▼
                         ┌─────────────────────┐
                         │   Sync Service       │  scoped to Jobs-portal/*
                         │  (FastAPI + APSched) │  writes raw emails → DB
                         └──────────┬───────────┘
                                    ▼
                         ┌─────────────────────┐
                         │  raw_emails (DB)     │
                         └──────────┬───────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │   Extraction Chain (LangChain) │  per portal template
                    │   email → array of listings    │
                    └───────────────┬────────────────┘
                                    ▼
                         ┌─────────────────────┐
                         │  job_listings (DB)   │
                         └──────────┬───────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │   Dedup (fuzzy match)          │
                    └───────────────┬────────────────┘
                                    ▼
              ┌────────────────────────────────────────┐
              │  Match Chain (LangChain, reasoning LLM)  │
              │  listings[] + user_profile → verdicts[]  │
              └───────────────────┬───────────────────────┘
                                    ▼
                         ┌─────────────────────┐
                         │  match_results (DB)  │
                         └──────────┬───────────┘
                                    ▼
                         ┌─────────────────────┐
                         │   REST API (FastAPI) │
                         └──────────┬───────────┘
                                    ▼
                         ┌─────────────────────┐
                         │   Frontend feed      │
                         └─────────────────────┘
```

Feature 3 (Application Tracker) runs a parallel branch: status-update emails →
a separate extraction chain → `applications` + `application_stage_history` tables →
kanban API/UI. Not built in Phase 1; schema included below so the DB doesn't need
breaking migrations later.

Feature 1 (automated classification) sits upstream of the Sync Service in the future
— for now, Gmail labels do this job manually, so Sync Service just reads by label.

## 2. Data model (PostgreSQL)

```sql
-- User profile: static input to every match call
CREATE TABLE user_profile (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    current_role        TEXT,
    years_experience     NUMERIC,
    core_skills          TEXT[] NOT NULL DEFAULT '{}',
    seniority_level       TEXT CHECK (seniority_level IN ('junior','mid','senior','staff')),
    preferred_work_setting TEXT[] NOT NULL DEFAULT '{}', -- remote | hybrid | onsite
    preferred_locations   TEXT[] NOT NULL DEFAULT '{}',
    min_salary            NUMERIC,
    salary_currency       TEXT,
    deal_breakers         TEXT[] NOT NULL DEFAULT '{}',
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Gmail sync bookkeeping, per label
CREATE TABLE gmail_sync_state (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    label_name        TEXT NOT NULL UNIQUE,       -- e.g. 'Jobs-portal/xing'
    last_history_id   TEXT,
    last_synced_at    TIMESTAMPTZ
);

-- Raw ingested emails (source of truth, immutable after insert)
CREATE TABLE raw_emails (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gmail_message_id TEXT NOT NULL UNIQUE,
    gmail_thread_id   TEXT,
    label             TEXT NOT NULL,              -- which Jobs-portal/* label
    source_portal     TEXT,                       -- indeed | xing | stepstone.de | ...
    sender            TEXT,
    subject           TEXT,
    received_at       TIMESTAMPTZ,
    body_cleaned      TEXT,                        -- signature/quote-stripped
    extraction_status TEXT NOT NULL DEFAULT 'pending', -- pending | done | failed
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Listings extracted from raw_emails (one email -> many listings, esp. digests)
CREATE TABLE job_listings (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    raw_email_id      UUID NOT NULL REFERENCES raw_emails(id),
    listing_index     INT NOT NULL,               -- position within the email
    title             TEXT,
    company           TEXT,
    location           TEXT,
    work_setting        TEXT,                      -- remote | hybrid | onsite | unspecified
    seniority_level      TEXT,
    skills_mentioned     TEXT[] NOT NULL DEFAULT '{}',
    salary_range          TEXT,
    platform_verdict      TEXT,                     -- e.g. "This is a bad match" (Indeed)
    job_url                TEXT,
    dedup_key               TEXT,                    -- normalized company+title+location
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (raw_email_id, listing_index)
);
CREATE INDEX idx_job_listings_dedup_key ON job_listings (dedup_key);

-- Match verdicts (one per listing, against the current profile)
CREATE TABLE match_results (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_listing_id        UUID NOT NULL REFERENCES job_listings(id),
    user_profile_id        UUID NOT NULL REFERENCES user_profile(id),
    match_score             NUMERIC NOT NULL,
    verdict                   TEXT NOT NULL CHECK (verdict IN
                                ('strong_match','possible_match','weak_match','mismatch')),
    matched_criteria           TEXT[] NOT NULL DEFAULT '{}',
    mismatched_criteria         TEXT[] NOT NULL DEFAULT '{}',
    reasoning                    TEXT,
    user_feedback                  TEXT CHECK (user_feedback IN ('up','down',NULL)),
    promoted_to_application_id     UUID,            -- nullable FK to applications, set on "apply"
    created_at                       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Feature 3 (built later, schema reserved now)
CREATE TABLE applications (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company           TEXT NOT NULL,
    role_title         TEXT NOT NULL,
    current_stage        TEXT NOT NULL,             -- applied | acknowledged | screening |
                                                      -- assessment | interview_scheduled |
                                                      -- interview_completed | offer | rejected | withdrawn
    next_action_date       TIMESTAMPTZ,
    recruiter_name            TEXT,
    recruiter_email            TEXT,
    created_at                   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE application_stage_history (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id    UUID NOT NULL REFERENCES applications(id),
    stage             TEXT NOT NULL,
    occurred_at        TIMESTAMPTZ NOT NULL,
    source_raw_email_id UUID REFERENCES raw_emails(id)
);

ALTER TABLE match_results
    ADD CONSTRAINT fk_promoted_application
    FOREIGN KEY (promoted_to_application_id) REFERENCES applications(id);
```

## 3. LangChain pipeline detail

Two independent chains, both using structured output (Pydantic models +
LangChain's `with_structured_output` or an equivalent parser), so output is always
validated JSON, never free text.

### 3.1 Extraction chain (per email → array of listings)

- Input: `raw_emails.body_cleaned`, `source_portal` (used to pick a portal-specific
  prompt/few-shot examples, since Indeed/Xing/StepStone templates differ)
- Output schema (Pydantic):

```python
class Listing(BaseModel):
    title: str | None
    company: str | None
    location: str | None
    work_setting: Literal["remote", "hybrid", "onsite", "unspecified"]
    seniority_level: Literal["junior", "mid", "senior", "unspecified"]
    skills_mentioned: list[str]
    salary_range: str | None
    platform_verdict: str | None
    job_url: str | None

class ExtractionResult(BaseModel):
    source_portal: str
    listings: list[Listing]
```

- Model: small/local extractor (Schemer-style or a grammar-constrained small LLM).
  Extraction is decode-only — no judgment, no profile awareness.
- Failure handling: if parsing fails or confidence is low, mark
  `raw_emails.extraction_status = 'failed'` and skip rather than guess.

### 3.2 Match chain (listings[] + profile → verdicts[])

- Input: all `job_listings` for one `raw_email_id` (batched, not one call per
  listing) + current `user_profile` row
- Output schema:

```python
class MatchVerdict(BaseModel):
    listing_index: int
    match_score: float
    verdict: Literal["strong_match", "possible_match", "weak_match", "mismatch"]
    matched_criteria: list[str]
    mismatched_criteria: list[str]
    reasoning: str

class MatchBatchResult(BaseModel):
    matches: list[MatchVerdict]
```

- Model: reasoning-capable (Gemma 4 E4B / Phi-4-mini-reasoning, or a hosted model if
  the on-device constraint is relaxed for this step) — still grammar-constrained to
  the schema above.
- `platform_verdict` (e.g. Indeed's own "bad match" label) is passed into the prompt
  as context, not treated as ground truth.

### 3.3 Dedup (non-LLM step, between extraction and matching)

- Normalize `company + title + location` (lowercase, strip legal suffixes like
  "GmbH"/"Corp", collapse whitespace) into `dedup_key`
- Fuzzy match (e.g. trigram similarity via Postgres `pg_trgm`) against existing
  `job_listings.dedup_key` before inserting a new row — same role posted across
  multiple portals or re-sent in a later digest collapses to one listing

## 4. API surface (FastAPI, Phase 1 scope)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/auth/gmail/callback` | OAuth callback, store tokens |
| `POST` | `/sync/run` | Trigger incremental sync for scoped labels |
| `GET` | `/profile` | Fetch current user profile |
| `PUT` | `/profile` | Update user profile |
| `GET` | `/feed` | Ranked recommendation feed, filterable by portal/verdict |
| `POST` | `/matches/{id}/feedback` | Thumbs up/down on a match |
| `POST` | `/matches/{id}/promote` | Mark as applied → creates `applications` row (Phase 3) |

Phase 3 adds `/applications`, `/applications/{id}/stage` etc. — not built yet, listed
here so the API design stays consistent when that phase starts.

## 5. Sync strategy

- Scope: only `Jobs-portal/*` labels (and their children) for Feature 2.
- Incremental: Gmail `history.list` per label, using `gmail_sync_state.last_history_id`
  as the watermark — avoids re-fetching the whole label every run.
- Backfill: one-time job per label for historical mail, rate-limited (batch + delay)
  given the volume skew (StepStone.de: 1,748 vs justjoin.it: 4).
- Body cleaning happens at ingest time (strip quoted history/signatures) before
  storing `body_cleaned`, so extraction always works on the same clean input shape.

## 6. Error handling

- Extraction failures: stored as `extraction_status = 'failed'` on `raw_emails`,
  surfaced in a small "needs review" list rather than silently dropped — useful
  signal that a portal's template changed.
- Match chain failures: retried once with backoff; if still failing, listing stays
  unscored rather than defaulting to a guessed verdict.
- All LangChain structured-output calls should validate against the Pydantic schema
  before writing to DB — a schema-invalid response is treated as a failure, not
  coerced.

## 7. Security

- Gmail OAuth tokens encrypted at rest (e.g. `pgcrypto` or app-level encryption
  before insert).
- Read-only Gmail scope (`gmail.readonly` + label metadata) — no send/modify scope
  needed for Phase 1.
- Single-user for now; no auth layer beyond the Gmail OAuth session needed until
  multi-user is in scope (explicitly out of scope per `planning.md`).

## 8. Deployment (MVP)

- Single FastAPI service + Postgres, run locally or on one small VM/container.
- Scheduled sync via APScheduler in-process for Phase 1 (simple); move to a proper
  job queue (Celery/arq + Redis) only if sync volume or reliability needs outgrow the
  in-process scheduler.
- No need to split extraction/match into separate services at this scale — keep them
  as chains within the same FastAPI app, split later only if latency or scaling
  actually demands it.
