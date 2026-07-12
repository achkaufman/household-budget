# Household Budget App — Phase 1 Plan

## Overview

A locally-run full-stack intelligent budgeting app. All data stays local (SQLite + local file storage). The only remote dependency is Azure OpenAI for LLM features.

---

## Architecture

```
┌─────────────────┐     REST      ┌──────────────────┐     SQL      ┌──────────┐
│  React Frontend │ ────────────► │  FastAPI Backend  │ ──────────► │  SQLite  │
│  (Vite + TS)    │               │  (Python)         │             │   DB     │
└─────────────────┘               └────────┬─────────┘             └──────────┘
                                           │  function calls
                                           ▼
                                  ┌──────────────────┐    tools     ┌──────────┐
                                  │   Azure OpenAI   │ ◄──────────► │  MCP     │
                                  │   (remote LLM)   │              │  Server  │
                                  └──────────────────┘              └──────────┘
```

**Component responsibilities:**
- **Frontend**: UI pages, calls backend REST API only
- **Backend**: Business logic, file I/O, LLM orchestration, DB access
- **MCP Server**: Exposes budget tools as MCP-compatible tools; backend acts as MCP host; also allows direct connection from an MCP-capable chat client (e.g. Claude Desktop)
- **SQLite**: Single source of truth for transactions
- **Local file storage**: Raw uploaded CSVs retained for audit/reprocessing
- **Azure OpenAI**: Powers categorization, recurring detection, and recommendations via function calling

---

## Project Structure

```
household-budget/
├── frontend/             # React + Vite + TypeScript
├── backend/              # Python FastAPI
│   ├── routers/          # Route handlers
│   ├── services/         # Business logic
│   ├── db/               # SQLite access layer
│   └── llm/              # Azure OpenAI client
├── mcp-server/           # Python MCP server
├── data/
│   ├── db/budget.db      # SQLite file (gitignored)
│   ├── uploads/          # Uploaded CSV statements (gitignored)
│   └── logs/             # Log files (gitignored)
├── .env                  # Secrets — never committed
├── .env.example          # Committed template (no values)
└── README.md
```

---

## UI: React Frontend

**Stack**: React 18, Vite, TypeScript, React Router, TanStack Query (data fetching), Tailwind CSS

### Pages

| Route | Page | Content |
|---|---|---|
| `/` | **Dashboard** | Tile 1: Current month expenses by category; Tile 2: Next 3–5 upcoming recurring transactions; Tile 3: Top 3–5 savings recommendations |
| `/transactions` | **Transactions** | Paginated list of all transactions, newest first. Columns: Date, Account, Description, Category, Amount, Recurring flag |
| `/upload` | **Upload Statement** | File picker (CSV only) + Import button; import status/progress feedback |

### Engineering Notes
- **Security**: Validate file type client-side (`.csv` only) before sending to backend. Never display raw HTML from API responses.
- **Env config**: All API base URL config via Vite env vars (`VITE_API_BASE_URL`), defaulting to `http://localhost:8000`.
- **Tests**: Vitest + React Testing Library; test each page component and key interactions.
- **CI/CD readiness**: `npm run build` produces a static bundle deployable behind any web server.

---

## Backend: FastAPI (Python)

**Stack**: Python 3.11+, FastAPI, Pydantic v2, `sqlite3` (stdlib), `httpx` (Azure OpenAI calls), `python-multipart` (file upload)

### Endpoints

| Method | Path | Description | LLM? |
|---|---|---|---|
| `GET` | `/api/transactions` | List transactions (paginated, newest first) | No |
| `POST` | `/api/upload` | Receive CSV, save to `data/uploads/` | No |
| `POST` | `/api/import` | Parse uploaded CSV, deduplicate, insert new rows | Calls categorize |
| `GET` | `/api/dashboard` | Returns summary, upcoming recurrings, recommendations | Calls recommendations |
| `POST` | `/api/recurring` | Trigger recurring transaction detection | Yes |

**Layer placement of intelligent features:**
- **Categorization** → triggered automatically during import; LLM call made in backend `services/categorize.py`, result written to DB if confidence is high; flagged (`needs_review=1`) if medium; left uncategorized if low
- **Recurring detection** → backend service `services/recurring.py` calls LLM on demand; updates `recurring=1` in DB
- **Recommendations** → backend service `services/recommendations.py` calls LLM on demand; results returned to dashboard (not persisted)

### Engineering Notes
- **Security**: Validate uploaded files — check MIME type and extension; reject anything that isn't CSV. Use `secure_filename` pattern to prevent path traversal. Use parameterized queries exclusively (no string interpolation in SQL). Restrict CORS to `http://localhost:5173` only. Store Azure OpenAI key in env var, never in code.
- **Secrets**: Load config via `pydantic-settings` from `.env`; fail fast on startup if `AZURE_OPENAI_API_KEY` or endpoint is missing.
- **Observability**: Structured JSON logging (stdlib `logging` + custom JSON formatter) to `data/logs/app.log` with rotating file handler (10 MB, 5 backups). Log every request (method, path, status, duration) via middleware. Log every LLM call (model, prompt tokens, completion tokens, duration, confidence result). Add a `correlation_id` (UUID) to each request, propagate to all log entries for that request.
- **CI/CD readiness**: `requirements.txt` pinned. App is stateless (all state in SQLite/filesystem) — ready to containerize with a Dockerfile later. Environment-based config (no hardcoded paths).
- **Tests**: `pytest` + `httpx` test client. Unit test CSV parsing and deduplication logic. Mock Azure OpenAI with `unittest.mock`. Run with `pytest --cov`.
- **Performance**: Enable SQLite WAL mode on startup. Index `transactions(date)` and `transactions(category)`. Paginate `/api/transactions` (default 50 rows).

---

## LLM: Azure OpenAI

**Deployment**: `gpt-4o` (or `gpt-4o-mini` for cost savings) on Azure OpenAI. Called from backend via REST using `httpx`.

**Prompt design** (one system prompt per feature):

| Feature | Context passed to LLM | Output |
|---|---|---|
| Categorize | Last 20 transactions (description + category) as few-shot examples; new transaction description + amount | `{ "category": "...", "confidence": "high|medium|low" }` |
| Recurring detection | Last 90 days of transactions grouped by description | `[ { "description": "...", "recurring": true } ]` |
| Recommendations | Last 30 days of transactions grouped by category with totals | `[ { "category": "...", "recommendation": "..." } ]` (top 5) |

**Layer placement**: LLM is called only from the backend services layer. MCP server tools wrap these same services.

### Engineering Notes
- **Security**: Never send PII beyond what's needed (description + amount + category only — no account numbers). Validate and sanitize all LLM responses before writing to DB; never execute LLM-generated code.
- **Resilience**: Wrap LLM calls in try/except; on failure, log and continue (import proceeds without categorization). Set request timeout of 30 seconds.

---

## MCP Server

**Stack**: Python, `mcp` SDK (`pip install mcp`)

**Purpose**: Exposes budget capabilities as MCP tools. Backend acts as the MCP host when orchestrating LLM calls. Also allows a chat client (e.g. Claude Desktop) to query the budget directly.

### Tools

| Tool | Description |
|---|---|
| `list_transactions` | Returns recent transactions from SQLite (accepts optional `limit`, `category`, `since` filters) |
| `import_statement` | Given a CSV file path, parse, deduplicate, and store transactions |
| `categorize_transaction` | Given description + amount, return suggested category + confidence |
| `identify_recurring_transactions` | Scan recent transactions and mark recurring ones |
| `get_recommendations` | Return top 3–5 spending reduction recommendations |

### Engineering Notes
- **Security**: MCP server only listens on localhost. Validate all tool input parameters with Pydantic before executing.
- **CI/CD readiness**: Can be exposed as a stdio or SSE transport; stdio is simplest for local dev.

---

## Database: SQLite

**File**: `data/db/budget.db`

### Schema

```sql
CREATE TABLE transactions (
    id          TEXT    PRIMARY KEY,   -- SHA-256(date||account||description||amount) or bank-provided ID
    date        TEXT    NOT NULL,      -- ISO 8601: YYYY-MM-DD
    account     TEXT,                  -- Last 4 digits of account number
    description TEXT    NOT NULL,
    category    TEXT,                  -- NULL until categorized
    amount      REAL    NOT NULL,      -- Negative = debit, positive = credit
    recurring   INTEGER NOT NULL DEFAULT 0,  -- 1 = recurring
    needs_review INTEGER NOT NULL DEFAULT 0, -- 1 = medium-confidence categorization
    created_at  TEXT    NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_transactions_date     ON transactions(date);
CREATE INDEX idx_transactions_category ON transactions(category);
```

**Deduplication**: On import, compute the same ID hash from CSV data; `INSERT OR IGNORE` handles duplicates silently.

**Transaction ID generation**: If the bank CSV includes a transaction ID column, use it. Otherwise compute `sha256(date + account + description + str(amount))` truncated to 16 hex chars.

### Engineering Notes
- **Security**: All queries use parameterized statements. The DB file lives outside the source tree (`data/`) and is gitignored.
- **Startup**: Enable WAL mode (`PRAGMA journal_mode=WAL`) and foreign keys (`PRAGMA foreign_keys=ON`) on every connection open.
- **Portability**: Keep the DB access layer behind an interface (`db/transactions.py`) so it can be swapped for Postgres later with minimal changes.

---

## File Storage

**Location**: `data/uploads/` (local, gitignored)

- Uploaded CSVs are saved with a sanitized, timestamped filename: `{YYYYMMDD_HHMMSS}_{sanitized_original_name}.csv`
- Files are retained after import for audit/reprocessing purposes
- No automatic cleanup in Phase 1

### Engineering Notes
- **Security**: Validate content type and file extension server-side. Strip directory components from filename. Restrict upload size to 10 MB.

---

## Logging

**Destination**: `data/logs/app.log` — rotating file handler (10 MB per file, 5 backups retained)

**Format**: Structured JSON per line, fields: `timestamp`, `level`, `correlation_id`, `logger`, `message`, plus any extra context (endpoint, duration, etc.)

**What to log**:
- Every HTTP request/response (method, path, status code, duration ms) — INFO
- Every LLM call (feature, model, tokens, duration, confidence) — INFO
- Import results (rows parsed, duplicates skipped, rows inserted) — INFO
- Errors and exceptions with full tracebacks — ERROR

**What NOT to log**: Raw transaction descriptions or amounts at DEBUG level in production; no API keys; no full file paths containing user data.

> Logs are kept separate from SQLite. No log table in the DB — log files are sufficient for a local MVP.

### Engineering Notes
- **CI/CD readiness**: JSON-formatted logs are ready to be shipped to a log aggregator (e.g. Azure Monitor, Loki) when deployed.

---

## Non-Functional Requirements

| Requirement | Approach |
|---|---|
| **Availability** | Local app; no HA needed. SQLite WAL mode prevents reader/writer contention. |
| **Performance** | Indexed queries; paginated list endpoint; LLM calls are async (don't block import). |
| **Maintainability** | Small focused modules (routers / services / db / llm). Pydantic models for all data shapes. Linting enforced (`ruff` for Python, `eslint` for TS). |
| **Scalability** | DB access layer abstracted; ready to swap SQLite for Postgres. Backend is stateless; ready to containerize. |
| **Data integrity** | SHA-256 based deduplication. SQLite transactions for multi-row writes. |
| **Privacy** | All data local. Only transaction descriptions/amounts leave the machine (to Azure OpenAI). Never log sensitive values. |

---

## Documentation

**README.md** (top-level) should cover:
1. Prerequisites (Python 3.11+, Node 20+)
2. Setup: `cp .env.example .env` and fill in Azure OpenAI values
3. Install & run backend: `pip install -r requirements.txt && uvicorn backend.main:app`
4. Install & run frontend: `cd frontend && npm install && npm run dev`
5. Install & run MCP server: `cd mcp-server && pip install -r requirements.txt && python server.py`
6. How to import a statement (upload CSV via UI)
7. How to connect MCP server to Claude Desktop (optional)

---

## CI/CD Readiness Notes

These apply now, before any pipeline exists:

- **No secrets in source**: `.env` gitignored; `.env.example` committed with placeholder values
- **Reproducible builds**: `requirements.txt` pinned with exact versions; `package-lock.json` committed
- **Linting as a gate**: `ruff check .` (Python) and `npm run lint` (TS) should pass before any merge
- **Test command**: `pytest` and `npm test` both runnable with no setup beyond install
- **Containerization path**: Structure backend and MCP server so a `Dockerfile` can be added at the root of each with minimal changes (no hardcoded local paths, config via env vars)
- **Branch strategy**: `main` = stable; feature branches named `feature/<short-description>`
