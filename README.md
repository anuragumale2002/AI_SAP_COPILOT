# SAP Project Copilot MVP

Competition-ready vertical slice for turning a Business Requirements Document (BRD) into a reviewable requirement baseline and a template-driven SAP Functional Specification Document (FSD).

This README covers what the application does and how to install and run it **locally** on macOS, Linux, or Windows.

---

## Table of contents

1. [What works](#what-works)
2. [Prerequisites](#prerequisites)
3. [Repository layout](#repository-layout)
4. [Local setup overview](#local-setup-overview)
5. [Create and use a Python virtual environment](#create-and-use-a-python-virtual-environment)
6. [Configure environment variables](#configure-environment-variables)
7. [Run the backend](#run-the-backend)
8. [Run the frontend](#run-the-frontend)
9. [Verify the stack](#verify-the-stack)
10. [Optional: Jira Cloud](#optional-jira-cloud)
11. [Run tests](#run-tests)
12. [Daily restart](#daily-restart)
13. [Troubleshooting](#troubleshooting)
14. [FSD generation architecture](#fsd-generation-architecture)
15. [Application architecture](#application-architecture)
16. [Important OpenAI settings](#important-openai-settings)
17. [API](#api)
18. [Security and packaging](#security-and-packaging)

---

## What works

1. Upload a PDF, DOCX, TXT, or Markdown BRD.
2. Extract page/paragraph-aware source chunks; DOCX tables are retained as source evidence.
3. When `OPENAI_API_KEY` is configured, analyze the **complete BRD once** into a persistent structured `BRDKnowledgeBase` containing process, scope, stakeholders, data, screens, integrations, rules, messages, NFRs, risks, acceptance criteria, and tests.
4. Generate atomic, source-linked requirements and send them through human approval/rejection.
5. Build the FSD from **three explicit inputs**: the versioned FSD template contract, persisted BRD knowledge, and the approved requirement baseline.
6. Validate grounding, completeness, traceability, BRD-context coverage, and template limits before rendering.
7. Export DOCX/PDF FSD output plus delivery backlog, technical design, test pack, traceability, and AI Companion artifacts.

Without an API key, a local demonstration path remains available for development. With `OPENAI_API_KEY`, the application performs full BRD knowledge extraction, requirement intelligence, and model-backed FSD design. Responses use `store=False`; model/token usage is recorded locally.

---

## Prerequisites

Install these **before** creating a virtual environment or running `npm install`.

| Tool | Recommended version | Why it is needed |
| --- | --- | --- |
| **Python** | 3.11 or 3.12 (3.10+ should work) | FastAPI backend, PDF/DOCX parsing, SQLAlchemy |
| **pip** | Comes with Python | Installs Python packages into the venv |
| **Node.js** | 20 LTS or 22 LTS | Next.js frontend |
| **npm** | Comes with Node.js | Installs frontend packages |
| **Git** | Latest | Clone / pull the repository |
| **A code editor** | Optional | VS Code, Cursor, or similar |

### Optional but useful

- **OpenAI API key** — required for real BRD analysis, requirement intelligence, FSD design, and AI Companion. Leave it empty for the local demo extractor.
- **PostgreSQL** — only if you do not want SQLite. SQLite is the zero-setup default.
- **Jira Cloud** email + API token — only if you want backlog/test-case sync.

### Check that tools are installed

**macOS / Linux:**

```bash
python3 --version
pip3 --version
node --version
npm --version
git --version
```

**Windows (PowerShell):**

```powershell
python --version
pip --version
node --version
npm --version
git --version
```

Expected roughly:

- Python `3.11.x` or `3.12.x`
- Node `v20.x` or `v22.x`

If `python` is not found on macOS/Linux, use `python3`. If `python` is not found on Windows, install Python from [python.org](https://www.python.org/downloads/) and tick **Add Python to PATH**.

### Install missing tools

**macOS (Homebrew):**

```bash
brew install python@3.12 node git
```

**Ubuntu / Debian:**

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip nodejs npm git
```

Prefer Node 20+ from [nodejs.org](https://nodejs.org/) or nvm if the distro package is older.

**Windows:**

1. Install Python 3.12 from [python.org](https://www.python.org/downloads/) (enable PATH).
2. Install Node.js LTS from [nodejs.org](https://nodejs.org/).
3. Restart the terminal so `python` and `node` are on PATH.

---

## Repository layout

Work from the **project root** (the folder that contains `apps/`, `docs/`, and this `README.md`).

```text
.
├── apps/
│   ├── api/                 # FastAPI backend
│   │   ├── app/             # Application source
│   │   ├── templates/fsd/   # Versioned FSD contracts
│   │   ├── tests/           # Pytest suite
│   │   ├── requirements.txt
│   │   └── .env.example
│   └── web/                 # Next.js frontend
│       ├── app/
│       ├── package.json
│       └── .env.local.example
├── docs/                    # Pipeline and FSD design notes
├── outputs/                 # Sample BRD and test guide
└── README.md
```

Two processes must run at the same time:

| Service | Directory | Default URL |
| --- | --- | --- |
| Backend API | `apps/api` | http://localhost:8000 |
| Frontend UI | `apps/web` | http://localhost:3000 |

Keep two terminal windows open: one for each service.

---

## Local setup overview

First-time setup:

1. Confirm prerequisites.
2. Create a Python virtual environment in `apps/api`.
3. Install Python dependencies into that venv.
4. Copy `.env.example` to `.env` and add an API key if you have one.
5. Install Node packages in `apps/web`.
6. Copy `.env.local.example` to `.env.local`.
7. Start the API, then start the web app.
8. Open http://localhost:3000.

---

## Create and use a Python virtual environment

A virtual environment (`venv`) isolates this project's Python packages from the rest of your machine. Create it **inside** `apps/api` so backend commands stay simple.

### Why use a venv

- Avoids installing FastAPI/SQLAlchemy globally.
- Matches the versions in `apps/api/requirements.txt`.
- Can be deleted and recreated without affecting other projects.

### macOS / Linux

```bash
cd apps/api

python3 -m venv .venv
```

Activate it:

```bash
source .venv/bin/activate
```

Your prompt should show `(.venv)`. Confirm the venv Python and pip:

```bash
which python
python --version
pip --version
```

Upgrade pip, then install dependencies:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Leave the venv active while you run the API. To deactivate later:

```bash
deactivate
```

### Windows (PowerShell)

```powershell
cd apps/api

python -m venv .venv
```

If execution policy blocks activation:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Activate:

```powershell
.\.venv\Scripts\Activate.ps1
```

Or:

```powershell
.\.venv\Scripts\activate
```

Your prompt should show `(.venv)`. Then:

```powershell
python -m pip install --upgrade pip
pip install -r requirements.txt
```

If `Activate.ps1` is blocked, you can still start the API without activating:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --port 8000
```

### Windows (Command Prompt)

```bat
cd apps\api
python -m venv .venv
.venv\Scripts\activate.bat
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### Recreate the venv

If installs fail or packages look mixed with the system Python:

```bash
# macOS / Linux
deactivate
rm -rf .venv
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

```powershell
# Windows PowerShell
deactivate
Remove-Item -Recurse -Force .venv
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Do **not** commit `.venv`. It is gitignored and regenerated locally.

Packages installed from `requirements.txt` include FastAPI, Uvicorn, SQLAlchemy, Pydantic Settings, pypdf, python-docx, Pillow, ReportLab, the OpenAI SDK, and PostgreSQL driver support (`psycopg`).

---

## Configure environment variables

Never commit `.env` or `.env.local`. Copy the example files and edit the copies.

### Backend — `apps/api/.env`

From `apps/api` (venv does not need to be active for this copy):

```bash
# macOS / Linux
cp .env.example .env
```

```powershell
# Windows PowerShell
Copy-Item .env.example .env
```

Open `apps/api/.env` and set at least:

| Variable | Required | Notes |
| --- | --- | --- |
| `DATABASE_URL` | No | Default `sqlite:///./sap_copilot.db` creates `sap_copilot.db` in `apps/api`. For PostgreSQL use `postgresql+psycopg://USER:PASSWORD@HOST:5432/DBNAME`. |
| `OPENAI_API_KEY` | For live AI | Leave empty for demo extraction. |
| `OPENAI_BRD_MODEL` | No | Full-document BRD understanding. |
| `OPENAI_REQUIREMENTS_MODEL` | No | Atomic requirement extraction. |
| `OPENAI_FSD_MODEL` | No | FSD synthesis. |
| `CORS_ORIGINS` | Yes for local UI | Must include the frontend origin, default `http://localhost:3000`. |
| `ARTIFACT_DIR` | No | Default `generated`. |
| `JIRA_*` | No | Only for Jira Cloud sync. |

SQLite needs no extra install. Tables are created on API startup.

### Frontend — `apps/web/.env.local`

```bash
# macOS / Linux
cd apps/web
cp .env.local.example .env.local
```

```powershell
# Windows PowerShell
cd apps\web
Copy-Item .env.local.example .env.local
```

Default:

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
```

Change this only if the API listens on another host or port. Restart `npm run dev` after editing `.env.local`.

---

## Run the backend

From `apps/api`, with the venv **active**:

```bash
uvicorn app.main:app --reload --port 8000
```

Equivalent if the venv is not activated:

```bash
# macOS / Linux
.venv/bin/python -m uvicorn app.main:app --reload --port 8000
```

```powershell
# Windows
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --port 8000
```

`--reload` restarts the server when Python files change. Leave this terminal running.

Useful URLs:

| URL | Purpose |
| --- | --- |
| http://127.0.0.1:8000/health | Health and pipeline/model summary |
| http://127.0.0.1:8000/docs | Interactive OpenAPI UI |
| http://127.0.0.1:8000/redoc | Alternative API docs |

`GET /` returning 404 is expected. Use `/health` or `/docs`.

On startup the API creates `uploads/` and `generated/` under `apps/api` if they do not exist.

---

## Run the frontend

In a **second** terminal (venv is not required):

```bash
cd apps/web
npm install
npm run dev
```

`npm install` is only required the first time, after `package.json` changes, or after deleting `node_modules`.

Then open **http://localhost:3000**.

| Script | Purpose |
| --- | --- |
| `npm run dev` | Local development with hot reload |
| `npm run build` | Production build |
| `npm start` | Serve a production build (`npm run build` first) |
| `npm run lint` | Next.js lint |

If port 3000 is already in use, Next.js prints another URL (often 3001). If you use that URL, add it to backend `CORS_ORIGINS` (comma-separated) and restart the API.

Example:

```env
CORS_ORIGINS=http://localhost:3000,http://localhost:3001
```

---

## Verify the stack

1. Backend health: http://127.0.0.1:8000/health  
   - No OpenAI key → `demo` mode.  
   - Key present → `openai` mode.
2. Frontend: http://localhost:3000 loads the UI.
3. Optional smoke test from [`outputs/TEST-GUIDE.md`](outputs/TEST-GUIDE.md):
   - Project name: `S/4HANA Order-to-Cash Improvement`
   - File: `outputs/sample-sap-order-to-cash-brd.txt`
   - Click **Analyze BRD**, review requirements, then **Approve baseline**.

---

## Optional: Jira Cloud

Jira is not required to run locally.

In `apps/api/.env`:

```env
JIRA_URL=https://your-domain.atlassian.net
JIRA_EMAIL=your-email@example.com
JIRA_API_TOKEN=
JIRA_PROJECT_KEY=GP
```

Create an Atlassian API token in your Atlassian account settings. Restart the API after changing these values.

---

## Run tests

From `apps/api` with the venv active:

```bash
pip install pytest
pytest
```

Or:

```bash
.venv/bin/python -m pip install pytest
.venv/bin/python -m pytest
```

Tests live in `apps/api/tests/`. They do not start the Next.js app.

---

## Daily restart

After the first-time install, a typical session is:

**Terminal 1 — API**

```bash
cd apps/api
source .venv/bin/activate          # Windows: .\.venv\Scripts\Activate.ps1
uvicorn app.main:app --reload --port 8000
```

**Terminal 2 — web**

```bash
cd apps/web
npm run dev
```

Then open http://localhost:3000.

---

## Troubleshooting

| Symptom | What to try |
| --- | --- |
| `python: command not found` | Use `python3`, or reinstall Python with PATH enabled. |
| `No module named venv` | Install `python3-venv` (Debian/Ubuntu) or a full Python installer. |
| `pip` installs into system Python | Activate `.venv` first; `which python` should point at `apps/api/.venv`. |
| `uvicorn: command not found` | Venv not active, or `pip install -r requirements.txt` was skipped. Use `.venv/bin/python -m uvicorn ...`. |
| Frontend cannot reach API | Confirm the API is on port 8000, `NEXT_PUBLIC_API_URL=http://localhost:8000`, and `CORS_ORIGINS` includes the UI origin. Restart both processes. |
| Port 8000 or 3000 in use | Stop the other process, or change the port and update `.env` / `.env.local` / `CORS_ORIGINS`. |
| Empty or failed analysis | Start with `outputs/sample-sap-order-to-cash-brd.txt`. Demo mode without a key is limited compared to OpenAI. |
| `OPENAI_*` errors | Check the key, model names, and that `.env` is in `apps/api` (not the repo root). |
| SQLite lock / stale data | Stop the API. `sap_copilot.db` is local and gitignored; delete it only if you want a clean database. |
| `node-gyp` / npm errors | Use Node 20+. Delete `apps/web/node_modules` and `package-lock.json` only if needed, then `npm install` again. |
| PowerShell activation blocked | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`, or call `.venv\Scripts\python.exe` directly. |

---

## FSD generation architecture

```text
Uploaded BRD
    |
    +--> Local source extraction (page/chunk evidence)
    |
    +--> OpenAI full-document BRD understanding
             |
             v
       BRDKnowledgeBase (persisted)
             |
             +--> Requirement Intelligence --> Human Review --> Approved Baseline
             |
             +----------------------------------------------+
                                                            |
Versioned FSD Template Contract ----------------------------+
                                                            v
                                                   FSD Design Agent
                                                            |
                                                   Deterministic expansion
                                                            |
                                                     Release validation
                                                            |
                                                      DOCX / PDF
```

The FSD agent does **not** rely on conversation memory. The template contract is version-controlled locally, BRD knowledge is persisted in the database, and the approved requirements are stored as the human-reviewed baseline. This makes generation reproducible and cache-safe.

---

## Application architecture

- `apps/web`: Next.js App Router frontend.
- `apps/api`: FastAPI + SQLAlchemy backend.
- SQLite is the zero-setup default; set `DATABASE_URL` to PostgreSQL for production.
- `app/brd_schemas.py`: structured full-BRD knowledge contract.
- `app/services/brd_intelligence.py`: OpenAI full-document BRD extraction with local-chunk fallback.
- `app/services/requirement_intelligence.py`: atomic source-linked requirement extraction with coverage guards.
- `app/services/functional_design.py`: template + BRD knowledge + approved-requirement FSD synthesis.
- `templates/fsd/v3/template-contract.json`: machine-readable 17-section FSD contract and section-to-BRD source map.
- `app/services/fsd_validation.py`: grounding/traceability/release gate.
- `app/services/fsd_export.py`: deterministic DOCX/PDF renderer.

For design rationale, see [`docs/brd-context-pipeline.md`](docs/brd-context-pipeline.md) and [`docs/fsd-generation.md`](docs/fsd-generation.md).

---

## Important OpenAI settings

See `apps/api/.env.example`.

- `OPENAI_BRD_MODEL`: deep full-document BRD understanding.
- `OPENAI_REQUIREMENTS_MODEL`: repetitive atomic requirement extraction.
- `OPENAI_FSD_MODEL`: FSD synthesis from template + BRD knowledge + approved requirements.
- `OPENAI_BRD_USE_FILE_INPUT=true`: prefer direct file input so PDF page visuals/tables/mockups are available to the BRD extractor.
- `OPENAI_MAX_SOURCE_CHARACTERS` and `OPENAI_REQUIREMENTS_MAX_BATCHES`: **coverage guards**. The application fails explicitly instead of silently dropping later BRD content.

---

## API

- `POST /api/projects` — upload and analyze a BRD synchronously.
- `POST /api/projects/start` — upload and start background BRD/requirement analysis.
- `GET /api/projects/{id}` — fetch the review workspace.
- `GET /api/projects/{id}/brd-knowledge` — inspect the persisted structured BRD context used by downstream generation.
- `PATCH /api/requirements/{id}` — edit/review a requirement.
- `POST /api/projects/{id}/approve` — approve the reviewed baseline.
- `POST /api/projects/{id}/artifacts/{kind}` — generate a downstream artifact (`fsd`, `backlog`, `technical_design`, `test_cases`, `traceability`, or `companion`).
- `GET /api/projects/{id}/usage` — inspect recorded model/token usage.
- `GET /api/usage` — inspect all recorded local model/token usage.
- `GET /health` — health check and configured pipeline/model summary.

Interactive docs: http://127.0.0.1:8000/docs

---

## Security and packaging

Do not commit:

- `.env` / `.env.local` (secrets)
- user uploads
- generated artifacts
- local SQLite databases
- `.venv`
- `node_modules`
- `.next`

Keep using `.env.example` and `.env.local.example` as templates.

For what is included in a source distribution ZIP, see [`PACKAGING_NOTES.md`](PACKAGING_NOTES.md). A walkthrough of the UI after startup is in [`outputs/TEST-GUIDE.md`](outputs/TEST-GUIDE.md).
