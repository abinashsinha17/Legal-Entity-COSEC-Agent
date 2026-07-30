# Legal Entity COSEC Agent

## Part 1 — Company Setup (Data Ingestion & Entity Profiling)

Ingests messy, unstructured legal PDFs (Articles of Association, Board
Charters, Certificates of Incorporation) and converts them into a structured
"Corporate Profile" (`EntityProfile` JSON), stored in an entity registry
database and reviewable/correctable through a Streamlit UI.

```
[PDF Documents]  ->  [OCR / Chunking]   ->  [LLM Info Extraction]  ->  [Structured JSON]
 (AoA, COI, ...)      (pdfplumber +          (OpenAI gpt-4o,             (SQLite via
                       pypdfium2/            structured outputs,         SQLAlchemy —
                       Tesseract OCR         chunked per request         Entity Registry)
                       fallback)             to respect token limits)
```

## Part 2 — Calendar Building (Dynamic Compliance Mapper)

Once a profile is human-reviewed, evaluates it against a jurisdictional
rules database to compute dynamic filing deadlines (e.g. "14 days after the
incorporation anniversary"), and can push them to Outlook/Google Calendar.

```
[Entity Registry]  ->  [Agent Evaluates Rules DB]  ->  [Generates Action Dates]  ->  [Syncs to Calendar]
 (Jurisdiction,          (MCA, Companies House,          (Calculated deadlines,        (Outlook/Google
  incorporation_date,     timezone-aware)                 timezone-correct)             Graph APIs)
  last_agm_date)
```

Pitfalls this addresses:
- **LLM hallucinations on dates** — a compliance calendar can only be
  generated for an entity whose profile a human has explicitly reviewed and
  approved (`is_reviewed`/`reviewed_at` on the entity record).
- **Timezone discrepancies** — every rule carries its own IANA timezone
  (e.g. `Europe/London` for Companies House, `Asia/Kolkata` for the MCA), and
  "today"/anniversary dates are computed in that timezone, not the host
  server's local clock.

## Part 3 — Continuous Monitoring & Workflow Orchestration

A separate service (`monitoring/`) that watches the Part 2 compliance
calendar for approaching deadlines and reacts to real-time HR/board-
directory webhooks, activating a pre-configured legal workflow (document
drafting + human review) for each one.

```
[Compliance Calendar]  ->  [Scanner (APScheduler)]  ->  [Message Queue]  ->  [Orchestrator]  ->  [Draft + Approval]
 (via backend_client,        polls every                (this service's        worker thread       (UI, not Slack)
  read-only HTTP)             SCAN_INTERVAL_SECONDS)      own SQLite DB -
                                                           no Kafka/Redis)
```

It's built as its own FastAPI service, structurally independent of
`backend/` - it only talks to it over HTTP (same as the Streamlit UI does),
so it can be run, restarted, or scaled separately.

- **Scanner (`app/scanner.py`, the "Watcher")** - an APScheduler job that
  polls `GET /api/compliance-schedule` on an interval and queues a
  `DEADLINE_APPROACHING` message for any `PENDING` deadline within
  `DEADLINE_TRIGGER_THRESHOLD_DAYS` (a dedupe table stops it re-queuing the
  same deadline on every scan).
- **Webhook (`POST /api/webhooks/hr`)** - simulates an external HR/board-
  directory system (e.g. Workday/SAP) pushing a real-time event straight
  onto the queue, bypassing the scanner entirely - e.g. a director
  resignation immediately triggers a `File_Director_Removal_Form` workflow.
- **Message queue (`app/queue_bus.py`)** - **no Kafka/Redis**: a message is
  just a row in this service's own SQLite table. The Streamlit UI reads it
  directly for a live "message queue" panel, and it survives a restart the
  same way a broker's log would; a small in-process `queue.Queue` alongside
  it just avoids the orchestrator having to poll the DB in a tight loop.
- **Orchestrator (`app/orchestrator.py` + `app/workflows.py`, the "Doer")**
  - a background worker thread that consumes queued messages, drafts the
  relevant document (`app/document_generator.py` - template-based, branches
  per filing type), and raises an `ApprovalRequest` - the human-in-the-loop
  gate, surfaced in the UI instead of a Slack message. Failed workflow runs
  retry up to `MAX_WORKFLOW_RETRIES` before being marked `FAILED`.

## Project layout

```
backend/            FastAPI service: parsing, LLM extraction, compliance engine, REST API
  app/
    models/schemas.py            Pydantic schemas: EntityProfile, ComplianceRule/Event, etc.
    services/pdf_parser.py       Layout-aware PDF parsing + OCR fallback
    services/llm_extraction.py   Chunked structured extraction via OpenAI + merge
    services/entity_service.py  Part 1 pipeline orchestration
    services/compliance_engine.py  Part 2: pure rule-evaluation + timezone-aware date math
    services/compliance_service.py Part 2: rules DB, schedule persistence, review gate
    services/calendar_sync.py    Part 2: Outlook/Google Calendar push connectors
    data/compliance_rules_seed.json  Starter rules DB, loaded on first run
    db/                      SQLAlchemy models + session
    api/routes.py             REST endpoints
    main.py                   FastAPI app entrypoint
  tests/                      pytest suite (uses the sample PDFs in Data/)
monitoring/          FastAPI service: Part 3 scanner + message queue + orchestrator
  app/
    scanner.py                Watcher - polls the compliance calendar, queues deadlines
    queue_bus.py               The message queue itself (SQLite-backed, no Kafka/Redis)
    orchestrator.py             Worker thread that consumes queued messages
    workflows.py                 Per-filing-type drafting + approval-request logic
    document_generator.py         Template-based document drafting
    backend_client.py             Read-only HTTP client to the Part 1/2 backend
    routes.py                     REST endpoints (queue, approvals, webhooks, manual scan)
  tests/                       pytest suite (isolated in-memory DB per test)
ui/                Streamlit app: upload, review/correct profiles, compliance calendar,
                   monitoring & workflow approvals
Data/              Sample Companies House PDFs used for testing
```

## Prerequisites

- Python 3.11+ (developed against 3.14)
- An OpenAI API key with access to `gpt-4o` (or another structured-output-capable model)
- **Tesseract OCR** installed, for pages that are scanned images rather than
  real text (common for stamped Certificates of Incorporation). On Windows:
  `winget install --id UB-Mannheim.TesseractOCR`. No `poppler` install is
  needed — page rasterization uses `pypdfium2`, a pure-Python wheel.

## Setup

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\pip install -r requirements.txt

copy .env.example .env
# then edit .env: set OPENAI_API_KEY, and TESSERACT_CMD if tesseract isn't on PATH
```

## Running the backend API

```powershell
cd backend
.\.venv\Scripts\uvicorn app.main:app --reload --port 8000
```

- Interactive API docs: http://localhost:8000/docs
- Health check: http://localhost:8000/health

## Running the UI

The Streamlit app lives in `ui/` and talks to the backend over HTTP. It uses
the same virtual environment as the backend (streamlit is already listed in
`backend/requirements.txt`):

```powershell
cd ui
..\backend\.venv\Scripts\streamlit run app.py
```

Then open http://localhost:8501. If the backend isn't running on
`http://localhost:8000`, set the URL in the sidebar or via the `API_BASE_URL`
environment variable before launching Streamlit.

Workflow in the UI:
1. **Upload Documents** — drag in one or more PDFs (try the samples in `Data/`);
   watch each file's live progress bar through parsing/extraction.
2. **Entity Registry** — see ingestion status (`PENDING` → `PARSING` →
   `EXTRACTING` → `COMPLETE`/`FAILED`) and review status for every document.
3. Click **View** on any entity to see the extracted Corporate Profile,
   correct any field (directors, shareholders, PSC, registered office, etc.),
   download the original source PDF for audit, or trigger reprocessing.
4. On that same page, click **Mark as reviewed & approved** once you've
   checked the extracted data, then **Generate compliance schedule** to
   compute its filing deadlines.
5. **Compliance Calendar** (sidebar) — a master view of every entity's
   deadlines across the whole registry, plus the Rules Engine Database
   (view/add/delete jurisdictional filing rules).
6. **Monitoring & Workflows** (sidebar) — Part 3: run a deadline scan or
   simulate an HR webhook, watch the message queue, and approve/reject
   drafted filings. Requires the monitoring service to be running (below).

## Running the monitoring service (Part 3)

Uses the same virtual environment as the backend (it needs `apscheduler`
and `httpx` in addition to what's already there):

```powershell
cd monitoring
..\backend\.venv\Scripts\pip install -r requirements.txt

copy .env.example .env
# defaults work out of the box against a backend on localhost:8000

..\backend\.venv\Scripts\uvicorn app.main:app --port 8010
```

- Interactive API docs: http://localhost:8010/docs
- On startup it immediately runs one scan, then repeats every
  `SCAN_INTERVAL_SECONDS`.
- The Streamlit UI's **Monitoring & Workflows** page needs
  `MONITORING_API_BASE_URL` in `ui/.env` pointed at it (defaults to
  `http://localhost:8010/api`).

In the UI: click **Run scan now** to check the compliance calendar
immediately rather than waiting for the next interval, or use **Simulate a
real-time HR webhook** to fire a director-resignation event straight onto
the queue. Either one produces a row in the **Message queue** panel and,
once the orchestrator has drafted the filing, a card in **Approval
requests** where a COSEC officer downloads the draft and approves/rejects it.

## Running tests

```powershell
cd backend
.\.venv\Scripts\pytest -v
```

The monitoring service (Part 3) has its own suite, run from `monitoring/`
so it picks up `monitoring/.env` the same way the app does:

```powershell
cd monitoring
..\backend\.venv\Scripts\pytest -v
```

Tests exercise the real PDF parser (including the OCR fallback path) against
the sample documents in `Data/`, and mock the OpenAI call in the extraction
and API tests so the suite runs without live API access or cost.

## Notes on the sample data

`Data/companies_house_document.pdf` is the 47-page Certificate of
Incorporation + IN01 + PSC + Memorandum & Articles of Association bundle for
**BRE Europe UK Limited**. `Data/companies_house_document_2.pdf` is the
3-page Certificate of Incorporation on Change of Name, renaming it to
**Revantage Global Services UK Ltd**. Uploading both and letting extraction
run should produce a single consolidated profile per document; a future
"entity resolution" step (merging profiles that share a company number) is
natural follow-up work beyond Part 1.

## The compliance rules database

`backend/app/data/compliance_rules_seed.json` seeds two illustrative rules on
first run (UK Confirmation Statement, India MGT-7) — these are samples to
demonstrate the engine, **not vetted legal advice**; add/edit real rules via
the UI's Rules Engine Database tab or `POST /api/compliance-rules` before
relying on this for actual filings.

## Calendar sync

`push_to_outlook_calendar`/`push_to_google_calendar` (`calendar_sync.py`) push
events given a bearer `access_token` you've already obtained (e.g. via
Microsoft Graph Explorer or Google's OAuth Playground) - this project does
not implement its own OAuth sign-in flow. Paste the token into the "Sync
selected deadlines" section on an entity's page or the master calendar.

## Configuration reference (`backend/.env`)

Every value is **required** - `app/config.py` has no built-in defaults, so a
missing one fails loudly at startup rather than silently using a value
hidden in source code. `backend/.env.example` documents recommended
starting values for all of them; copy it to `.env` and fill it in. Never
commit `.env` (it holds your real API key).

| Variable                         | Purpose                                                        |
|-----------------------------------|-----------------------------------------------------------------|
| `OPENAI_API_KEY`                 | Used for structured extraction.                                |
| `OPENAI_MODEL`                   | e.g. `gpt-4o-2024-08-06`.                                       |
| `DATABASE_URL`                   | SQLAlchemy URL (a local SQLite file by default).                |
| `UPLOAD_DIR`                     | Where uploaded source PDFs are stored on disk.                  |
| `CORS_ORIGINS`                   | Comma-separated origins allowed to call the API.                |
| `TESSERACT_CMD`                  | *(optional)* full path to `tesseract.exe` if not on PATH.       |
| `OCR_MIN_CHARS_BEFORE_FALLBACK`  | Below this many extracted chars/page, fall back to OCR.        |
| `OCR_RASTER_DPI`                 | DPI used when rasterizing a page for OCR.                       |
| `LLM_CHUNK_MAX_CHARS`            | Max chars per LLM request; larger docs are chunked (see below). |
| `LLM_MAX_RETRY_ATTEMPTS`         | Retry attempts for a failed/invalid extraction call.            |
| `LLM_RETRY_WAIT_MIN_SECONDS`     | Exponential backoff lower bound between retries.                |
| `LLM_RETRY_WAIT_MAX_SECONDS`     | Exponential backoff upper bound between retries.                |
| `MIN_PLAUSIBLE_INCORPORATION_YEAR` | Rejects extracted dates with an implausible year.             |
| `RAW_TEXT_PREVIEW_CHARS`         | Length of the "raw parsed text" audit preview in the UI.        |

**On `LLM_CHUNK_MAX_CHARS`:** large documents (the 47-page bundle in `Data/`
is a real example) are split into chunks at page boundaries and extracted
+ merged separately, because a single oversized request can permanently
exceed a lower-tier OpenAI account's per-minute token budget - no amount of
retrying fixes a request that's simply bigger than the account's entire
per-minute allowance. The default (24000 chars, ≈6K tokens) is conservative
enough for low tiers; raise it for fewer/larger (faster) chunks if your
account has a higher TPM limit.

`ui/.env.example` documents the Streamlit app's own required settings
(`API_BASE_URL`, upload/detail poll intervals, `MONITORING_API_BASE_URL`) -
copy it to `ui/.env`.

## Configuration reference (`monitoring/.env`)

Same no-hardcoded-defaults philosophy as the backend - see
`monitoring/.env.example`.

| Variable                           | Purpose                                                          |
|-------------------------------------|-------------------------------------------------------------------|
| `BACKEND_API_URL`                  | Part 1/2 backend API base URL (read-only HTTP, no shared DB).    |
| `DATABASE_URL`                     | This service's own SQLite DB (queue, approvals, dedupe table).  |
| `STORAGE_DIR`                      | Where drafted documents are written.                             |
| `CORS_ORIGINS`                     | Comma-separated origins allowed to call this API (the UI).       |
| `SCAN_INTERVAL_SECONDS`            | How often the scanner re-checks the compliance calendar.         |
| `DEADLINE_TRIGGER_THRESHOLD_DAYS`  | Queue a deadline once it's within this many days (or overdue).   |
| `MAX_WORKFLOW_RETRIES`             | Retry attempts for a failed workflow run before marking it FAILED. |

## Known limitations

- **No database migrations.** Schema changes to already-existing tables
  (adding a column, etc.) aren't applied automatically - `Base.metadata.create_all()`
  only creates tables that don't exist yet. If you pull a change that alters
  the DB schema, delete `backend/storage/entities.db` (and re-upload/re-seed)
  rather than expecting it to migrate in place. A real deployment would want
  Alembic here.
- **No OAuth login flow** for calendar sync - see "Calendar sync" above.
- The seeded compliance rules are illustrative samples, not a maintained,
  legally-vetted ruleset.
- **Part 3's document drafter is template-based, not an LLM call** - it
  produces a deterministic placeholder document (clearly marked "DRAFT -
  requires officer review") so the scan → queue → orchestrate → approve
  pipeline is fully runnable and testable without an extra API key; swap
  `monitoring/app/document_generator.py`'s function bodies for a real LLM
  call without touching any of its callers.
- `monitoring/storage/workflow.db` has the same no-migrations limitation as
  `backend/storage/entities.db` (see above).
