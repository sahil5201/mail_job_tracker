# Mail Job Tracker

Connects to Gmail, reads job-related email, and turns it into structured, useful
views instead of a pile of unread labels:

- **Recommendation Feed** — job-board recommendation emails (Indeed, Xing, StepStone,
  Naukri, etc.) scored against your own skills/preferences, not keyword-matched.
- **Application Tracker** *(planned)* — a kanban of real applications built from
  status-update emails (Applied → Screening → Interview → Offer/Rejected).

See [`planning.md`](./docs/planning.md) for scope, phased roadmap, and open decisions.
See [`system_design.md`](./docs/system_design.md) for architecture, DB schema, and the
LangChain pipeline design.

Check [`High-Level System Design`](./docs/images/hlsd.png) for a visual representation.

## Feature status

| Feature | Status |
|---|---|
| Extractor & Classification | Deferred — handled manually via Gmail filters/labels for now |
| Recommendation Matcher | **Active — Phase 1 build** |
| Application Tracker | Planned — Phase 3 |

## Tech stack

- **Backend:** Python, FastAPI
- **Orchestration:** LangChain
- **Database:** PostgreSQL
- **Mail source:** Gmail API (OAuth2)

(Stack rationale — including why FastAPI over Hono.js/Bun — is in `planning.md #5`.)

## Project structure

```
.
├── app/
│   ├── main.py                # FastAPI app entrypoint
│   ├── api/                   # route handlers
│   │   ├── profile.py
│   │   ├── sync.py
│   │   └── feed.py
│   ├── chains/                # LangChain chains
│   │   ├── extraction.py      # email -> listings[]
│   │   └── matching.py        # listings[] + profile -> verdicts[]
│   ├── db/
│   │   ├── models.py          # SQLAlchemy / SQLModel models
│   │   └── session.py
│   ├── gmail/
│   │   ├── auth.py            # OAuth flow
│   │   └── sync.py            # incremental sync via history.list
│   └── schemas/               # Pydantic models shared by chains + API
├── alembic/                   # DB migrations
├── planning.md
├── system_design.md
├── readme.md
├── pyproject.toml
└── .env.example
```

## Setup

### Prerequisites
- Python 3.11+
- PostgreSQL 15+
- A Google Cloud project with the Gmail API enabled, OAuth2 credentials
  (client ID/secret) with `gmail.readonly` scope

### 1. Clone and install
```bash
git clone <repo-url>
cd mail-job-tracker
uv sync            # or: pip install -e .
```

### 2. Configure environment
```bash
cp .env.example .env
```
Fill in:
```
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/mail_job_tracker
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=http://localhost:8000/auth/gmail/callback
LLM_PROVIDER=...          # extraction/match model provider config
```

### 3. Set up the database
```bash
createdb mail_job_tracker
alembic upgrade head
```

### 4. Connect Gmail
```bash
uv run uvicorn app.main:app --reload
```
Visit `http://localhost:8000/auth/gmail/login`, authorize, and the app stores tokens
and begins tracking the configured `Jobs-portal/*` labels.

### 5. Set your profile
```bash
curl -X PUT http://localhost:8000/profile -H "Content-Type: application/json" -d '{
  "core_skills": ["React", "TypeScript", "Node.js"],
  "seniority_level": "mid",
  "preferred_work_setting": ["remote", "hybrid"],
  "preferred_locations": ["Bangalore", "Remote-India"],
  "deal_breakers": ["onsite-only outside my city"]
}'
```

### 6. Run a sync
```bash
curl -X POST http://localhost:8000/sync/run
```

### 7. View your feed
```bash
curl http://localhost:8000/feed?verdict=strong_match,possible_match
```
(A proper frontend for this comes once the API is stable — see `planning.md` Phase 1.)

## Running tests
```bash
uv run pytest
```

## Next steps
See `planning.md #7` for the phased roadmap — Phase 1 (this repo's current focus) is
the Recommendation Matcher on 2–3 portals before expanding further.
