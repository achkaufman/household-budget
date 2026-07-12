# Household Budget App — Phase 1 Plan (v2)

## Overview

A locally-run full-stack intelligent budgeting app. All user data stays local (SQLite + local file storage). The only remote dependency is Azure OpenAI for selected intelligence features.

This v2 keeps the same product scope as v1, with targeted architecture improvements for reliability, maintainability, and safer AI integration.

---

## Architecture

```
┌─────────────────┐   REST /api/v1   ┌───────────────────────────────┐
│  React Frontend │ ────────────────► │ FastAPI Backend (App Layer)   │
│  (Vite + TS)    │                   │ - Routers / Validation         │
└─────────────────┘                   │ - Query APIs                   │
                                      │ - Command APIs (enqueue jobs)  │
                                      └───────────────┬───────────────┘
                                                      │
                                      reads/writes    │ executes jobs
                                                      ▼
                                    ┌───────────────────────────────┐
                                    │  SQLite                       │
                                    │  - transactions               │
                                    │  - import_runs                │
                                    │  - recurring_patterns         │
                                    │  - app_jobs                   │
                                    └───────────────┬───────────────┘
                                                    │
                                                    ▼
                                    ┌───────────────────────────────┐
                                    │ Background Worker             │
                                    │ - CSV import + dedupe         │
                                    │ - Categorization pipeline     │
                                    │ - Recurring detection         │
                                    └───────────────┬───────────────┘
                                                    │
                                                    ▼
                                    ┌───────────────────────────────┐
                                    │ Azure OpenAI (remote)         │
                                    │ - Structured JSON output       │
                                    │ - Confidence + rationale tags  │
                                    └───────────────────────────────┘

Optional:
┌────────────────────┐
│ MCP Server (Python)│
│ wraps backend APIs │
└────────────────────┘
```

Component responsibilities:
- Frontend: user workflows and presentation only.
- Backend App Layer: API contracts, auth boundary (if added later), orchestration.
- Background Worker: long-running and retriable work.
- SQLite: source of truth plus job state.
- Azure OpenAI: assistive classification and recommendation text generation.
- MCP Server: optional integration layer for external AI clients.

---

## Key Improvements from v1

1. Added a lightweight background job model for imports and AI tasks to avoid blocking requests and improve resilience.
2. Split write workflows (commands) from read workflows (queries) to simplify scaling and testing.
3. Added structured schema evolution (migrations) and operational tables (`import_runs`, `app_jobs`) for traceability.
4. Made AI outputs schema-bound and policy-checked before persistence.
5. Upgraded recurring detection and recommendations to hybrid logic (deterministic first, LLM second) for lower cost and higher consistency.
6. Added explicit backup/restore and retention guidance for local reliability.

---

## Project Structure

```
household-budget/
├── frontend/                      # React + Vite + TypeScript
├── backend/
│   ├── routers/                   # HTTP route handlers (/api/v1)
│   ├── schemas/                   # Pydantic request/response models
│   ├── services/                  # Use cases (commands + queries)
│   ├── workers/                   # Background workers
│   ├── llm/                       # Azure OpenAI client + output validation
│   ├── db/
│   │   ├── repo/                  # Data access layer
│   │   ├── migrations/            # SQL migrations
│   │   └── connection.py
│   └── main.py
├── mcp-server/                    # Optional MCP adapter (no direct DB access)
├── data/
│   ├── db/budget.db               # SQLite file (gitignored)
│   ├── uploads/                   # Uploaded statements (gitignored)
│   ├── quarantine/                # Invalid/unparseable files (gitignored)
│   ├── exports/                   # Optional CSV exports (gitignored)
│   └── logs/                      # Log files (gitignored)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── contract/
├── .env                           # Secrets (gitignored)
├── .env.example
└── README.md
```

---

## UI: React Frontend

Stack: React 18, Vite, TypeScript, React Router, TanStack Query, Tailwind CSS.

### Pages

| Route | Page | Content |
|---|---|---|
| / | Dashboard | Current month spend by category, next 3-5 recurring transactions, top 3-5 savings opportunities |
| /transactions | Transactions | Paginated and filterable list, newest first |
| /upload | Upload Statement | CSV upload, validation result, import job status |
| /imports | Import History | Recent imports, rows inserted/skipped/errors, retry option |

### Engineering Notes
- Security: client validates type/size before upload, but server remains source of validation truth.
- Reliability UX: upload returns job id; UI polls import status endpoint.
- Accessibility: keyboard-friendly table filtering and upload flow.
- Tests: component tests plus one end-to-end flow (upload -> imported -> appears in transactions).
- CI/CD readiness: strict type checks and linting as pre-merge gates.

---

## Backend: FastAPI (Python)

Stack: Python 3.11+, FastAPI, Pydantic v2, sqlite3, httpx, python-multipart.

### API Endpoints (v1)

| Method | Path | Description |
|---|---|---|
| GET | /api/v1/transactions | List transactions (page/filter/sort) |
| GET | /api/v1/dashboard | Summary + recurrings + recommendations |
| POST | /api/v1/uploads | Save CSV and create import job |
| GET | /api/v1/imports/{import_id} | Import run details and status |
| GET | /api/v1/jobs/{job_id} | Generic job status for UI polling |
| POST | /api/v1/recurring/recompute | Enqueue recurring detection job |
| POST | /api/v1/recommendations/recompute | Enqueue recommendation refresh |

### Processing Model

- Synchronous request: validate input, persist metadata, enqueue job.
- Asynchronous worker: parse, normalize, dedupe, categorize, mark recurring, update aggregates.
- Status model: queued -> running -> succeeded or failed, with error message and retry count.

### Engineering Notes
- Security: strict CSV schema validation, filename sanitization, path traversal protection, parameterized SQL only.
- Reliability: idempotent job handlers keyed by import id; retry with capped attempts.
- Observability: structured logs + correlation id + job id in every processing log.
- API governance: all responses use typed Pydantic models and stable versioned routes.
- Maintainability: no business logic in routers; all use cases live in services/workers.

---

## LLM: Azure OpenAI

Deployment: Azure OpenAI GPT-4.1 or GPT-4o-mini depending on quality/cost target.

### LLM Usage Pattern

1. Deterministic pre-pass:
- Rule-based category mapping from known merchants and user overrides.
- Deterministic recurring detection based on cadence and amount tolerance.

2. LLM fallback:
- Only for uncertain cases.
- Structured JSON output with strict schema validation.

3. Write policy:
- High confidence: auto-write.
- Medium confidence: write suggestion and mark needs_review = 1.
- Low confidence: keep uncategorized and surface in review queue.

### Prompt Inputs and Outputs

| Feature | Input | Output |
|---|---|---|
| Categorize | recent labeled examples + candidate transaction | category, confidence, short_reason_code |
| Recurring assist | grouped transaction sequence candidates | recurring true/false + cadence hint |
| Recommendations | monthly aggregates + trend deltas | top recommendations with impact estimate |

### Engineering Notes
- Security: minimize outbound fields (no full account numbers, no user identifiers).
- Safety: reject malformed model output and fallback gracefully.
- Cost controls: token budget caps and request timeout (30s).
- Auditability: store model name and confidence for each AI-assisted decision.

---

## MCP Server

Stack: Python mcp SDK.

Purpose: optional integration endpoint for MCP-capable assistants. It should call backend APIs, not access SQLite directly.

### Tools

| Tool | Description |
|---|---|
| list_transactions | Query transactions with optional filters |
| import_statement | Trigger import from CSV path |
| categorize_transaction | Return suggested category and confidence |
| identify_recurring_transactions | Trigger recurring recompute |
| get_recommendations | Return current recommendations |
| get_import_status | Return latest import/job status |

### Engineering Notes
- Security: localhost-only binding for local mode.
- Contract stability: MCP tool inputs and outputs mirror backend schema contracts.

---

## Database: SQLite

File: data/db/budget.db

### Core Schema

```sql
CREATE TABLE transactions (
    id             TEXT PRIMARY KEY,
    date           TEXT NOT NULL,
    account_last4  TEXT,
    description    TEXT NOT NULL,
    normalized_desc TEXT,
    category       TEXT,
    amount         REAL NOT NULL,
    currency       TEXT NOT NULL DEFAULT 'USD',
    recurring      INTEGER NOT NULL DEFAULT 0,
    recurrence_key TEXT,
    needs_review   INTEGER NOT NULL DEFAULT 0,
    source_file    TEXT,
    created_at     TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at     TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE import_runs (
    import_id      TEXT PRIMARY KEY,
    file_name      TEXT NOT NULL,
    status         TEXT NOT NULL,
    rows_total     INTEGER NOT NULL DEFAULT 0,
    rows_inserted  INTEGER NOT NULL DEFAULT 0,
    rows_skipped   INTEGER NOT NULL DEFAULT 0,
    rows_failed    INTEGER NOT NULL DEFAULT 0,
    started_at     TEXT,
    finished_at    TEXT,
    error_message  TEXT
);

CREATE TABLE app_jobs (
    job_id         TEXT PRIMARY KEY,
    job_type       TEXT NOT NULL,
    status         TEXT NOT NULL,
    payload_json   TEXT,
    retry_count    INTEGER NOT NULL DEFAULT 0,
    created_at     TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at     TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_txn_date ON transactions(date);
CREATE INDEX idx_txn_category ON transactions(category);
CREATE INDEX idx_jobs_status ON app_jobs(status);
```

### ID and Dedup Strategy

- If bank transaction id exists, use it.
- Else compute stable id hash from normalized tuple:
  date + account_last4 + normalized_desc + amount + currency.
- Use INSERT OR IGNORE for idempotent imports.

### Engineering Notes
- Use migration scripts for all schema changes, including local development.
- Enable WAL mode and foreign_keys on each connection.
- Wrap each import in a transaction boundary for consistency.

---

## File Storage

Location: data/uploads/

- Store original upload with timestamped sanitized filename.
- Store parse failures in data/quarantine/ with reason metadata.
- Keep checksum per file to avoid accidental reprocessing.

### Engineering Notes
- Server-side file size limit (10 MB default).
- Validate delimiter/header shape before queuing import.

---

## Logging and Observability

Destination: data/logs/ with rolling JSON logs.

### Required Signals

- HTTP logs: method, route, status, duration, correlation id.
- Job logs: job id, job type, attempts, outcome.
- LLM logs: feature, model, latency, token usage, confidence bucket.
- Import metrics: rows parsed/inserted/skipped/failed.

### Storage Guidance

- Keep logs in files for local MVP.
- Do not store logs in primary SQLite transaction tables.
- Optional: periodic log compaction or retention deletion for disk control.

---

## Security Fundamentals

- Secrets only in env vars; never in repo.
- Input validation at all boundaries (HTTP, job payloads, MCP tools, model output).
- Least outbound data to LLM.
- Dependency scanning and credential scanning in CI.
- Localhost binding by default for backend and MCP services.

---

## Non-Functional Requirements

| Requirement | Approach |
|---|---|
| Availability | Local-first, recoverable job model, resumable imports |
| Performance | Indexed reads, pagination, background processing |
| Maintainability | Layered architecture, typed contracts, migration discipline |
| Scalability | Clear seams for swapping SQLite and externalizing workers later |
| Data integrity | Idempotent imports, transaction boundaries, stable dedupe ids |
| Privacy | Data remains local except minimal fields sent to Azure OpenAI |

---

## Testing Strategy

- Unit: CSV parser, dedupe id generation, categorization policy logic.
- Integration: API + SQLite + worker interaction tests.
- Contract: backend response schema tests and MCP tool contract tests.
- End-to-end: upload statement and verify dashboard totals + transaction visibility.
- Regression: fixed fixture CSVs for repeated import and edge-case coverage.

---

## Documentation

Top-level README should include:
1. Prerequisites (Python 3.11+, Node 20+)
2. Env setup from .env.example
3. Backend run command
4. Worker run command
5. Frontend run command
6. Optional MCP server run command
7. Upload/import workflow
8. Troubleshooting section (common CSV format issues)

---

## CI/CD Readiness Notes

- Reproducible builds with pinned dependencies.
- Lint/type/test commands standardized and scriptable.
- Database migration check included in CI.
- Security checks: dependency and secret scanning.
- Build artifacts separable for frontend, backend, worker, and MCP server.

---

## Minimal Delivery Sequence

1. Build schema + migration runner + repositories.
2. Implement upload -> import job lifecycle with status tracking.
3. Implement transactions and dashboard read APIs.
4. Add deterministic categorization and recurring logic.
5. Add LLM fallback for uncertain classification + recommendations.
6. Add MCP adapter and contract tests.
7. Complete README and local run scripts.
