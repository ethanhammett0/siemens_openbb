# Siemens Financial Services — OpenBB Workspace App

Custom backend and AI agent skills for building OpenBB Workspace dashboards tailored to Siemens Financial Services (SFS) structured finance workflows.

---

## What's In This Repo

This handoff packages three components:

**1. `structured_finance_app/`** — A production-ready FastAPI backend combining four internal dashboards (Markets Monitor, Healthcare Portfolio Manager, Salesforce Deal Tracker, SharePoint Document Viewer). Runs locally on port 7800 and connects directly to OpenBB Workspace at [pro.openbb.co](https://pro.openbb.co).

**2. `.claude/`** — AI agent configuration for Claude (and compatible agents). Includes a full library of OpenBB documentation cached locally, four validation scripts, and a suite of skills that give any AI coding agent a structured, battle-tested pipeline for building and extending OpenBB backends. These are the exact skills and docs used to build this app.

**3. `openbb_docs/`** — Offline copy of OpenBB Workspace developer documentation. Used as a local cache so agents don't need to fetch from the web during development.

---

## Repository Structure

```
siemens_openbb/
│
├── structured_finance_app/        # FastAPI backend — runs at port 7800
│   ├── main.py                    # App entry point, CORS config, router mounts
│   ├── widgets.json               # Widget definitions (object keyed by widget ID)
│   ├── apps.json                  # Dashboard layout config (array)
│   ├── data_loader.py             # CSV pre-loading at startup — no per-request I/O
│   ├── plotly_templates.py        # Shared Plotly dark/light theme helpers
│   ├── requirements.txt           # Python deps: fastapi, uvicorn, pandas, numpy, plotly
│   ├── run.bat                    # Windows one-click launcher
│   ├── AGENTS.md                  # Agent rules & critical OpenBB schema reference
│   ├── routes/
│   │   ├── markets.py             # Deal flow distribution, LTV scatter, raincloud, radar
│   │   ├── portfolio.py           # Healthcare RE assets, tenants, maturities, cash flows
│   │   ├── forms.py               # Deal entry, tranche tracking, account lookup
│   │   └── sharepoint.py          # Document browser and PDF viewer
│   ├── data/                      # Synthetic HRE portfolio CSVs
│   │   ├── hre_cashflows.csv
│   │   ├── hre_deals.csv
│   │   ├── hre_debt_service.csv
│   │   ├── hre_properties.csv
│   │   └── hre_tenants.csv
│   ├── forms_data/                # SFS pipeline, tranche, account, and RE asset CSVs
│   │   ├── sfs_pipeline_log.csv
│   │   ├── tranches_log.csv
│   │   ├── involved_accounts.csv
│   │   └── realestate_assets.csv
│   └── dummy_pdf/                 # Sample deal documents for SharePoint viewer
│       ├── Due Diligence/
│       └── Financials/
│
├── .claude/                       # AI agent configuration (Claude / Claude Code)
│   ├── docs/                      # Local OpenBB documentation cache
│   │   ├── INDEX.md               # Navigation guide — start here
│   │   ├── workspace-fundamentals.md
│   │   ├── odp-data-integration.md
│   │   ├── api-keys.md
│   │   ├── openbb-ai-sdk.md
│   │   ├── agent-patterns.md
│   │   ├── sse-protocol-models.md
│   │   ├── apps-configuration.md
│   │   ├── mcp-server.md
│   │   └── testing.md
│   ├── scripts/                   # Validation scripts (run before connecting to OpenBB)
│   │   ├── validate_widgets.py    # Validate widgets.json schema
│   │   ├── validate_apps.py       # Validate apps.json schema
│   │   ├── validate_endpoints.py  # Check all widget endpoints resolve
│   │   └── validate_app.py        # Full-app validation (widgets + apps + endpoints)
│   └── skills/
│       ├── openbb-app-builder/    # 7-phase pipeline: interview → build → validate → test
│       │   ├── SKILL.md
│       │   └── references/
│       │       ├── WIDGET-METADATA.md
│       │       ├── PARAMETER-OPTIMIZATION.md
│       │       ├── DASHBOARD-LAYOUT.md
│       │       ├── VALIDATE.md
│       │       ├── APP-INTERVIEW.md
│       │       ├── APP-PLANNER.md
│       │       ├── APP-TESTER.md
│       │       ├── ARCHITECTURE.md
│       │       └── OPENBB-APP.md
│       ├── api-discovery/         # Explore APIs before building
│       │   ├── SKILL.md
│       │   └── references/
│       │       ├── API-EXPLORER.md
│       │       ├── PERSONA-MAPPER.md
│       │       └── WIDGET-OPTIMIZER.md
│       ├── odp-backend/           # ODP credentials, data pre-loading, copilot agents
│       │   └── SKILL.md
│       └── improve-openbb-skill/  # Submit upstream fixes to OpenBB-finance
│           └── SKILL.md
│
└── openbb_docs/                   # Offline OpenBB developer documentation snapshot
    └── OpenBB-develop/
```

---

## Dashboards

| Dashboard | Route Module | Description |
|-----------|-------------|-------------|
| **Markets Monitor** | `routes/markets.py` | Deal flow distribution by interval/metric, LTV vs. spread scatter, raincloud density, radar benchmarking |
| **Healthcare Portfolio Manager** | `routes/portfolio.py` | Asset-level portfolio grid with row grouping, cash flow history, maturity schedule, allocation charts |
| **Salesforce / Deal Tracker** | `routes/forms.py` | Deal entry, tranche tracking, real estate assets, account lookup — backed by SFS pipeline CSVs |
| **SharePoint Viewer** | `routes/sharepoint.py` | Browse and preview deal documents and PDFs from SharePoint-style folder structures |

---

## Quick Start

**Prerequisites:** Python 3.10+

```bash
cd structured_finance_app
pip install -r requirements.txt
uvicorn main:app --reload --port 7800
```

Or on Windows, double-click `run.bat`.

**Connect to OpenBB Workspace:**

1. Open [pro.openbb.co](https://pro.openbb.co)
2. Settings → Data Connectors → Add Backend
3. Enter: `http://localhost:7800`
4. Click Validate — all four dashboards appear in the app gallery

---

## Running Validation Before Connecting

OpenBB Workspace fails silently on schema errors. Run these scripts first:

```bash
cd structured_finance_app

# Validate widgets.json structure
python ../.claude/scripts/validate_widgets.py widgets.json

# Validate apps.json structure
python ../.claude/scripts/validate_apps.py apps.json

# Check all widget endpoints resolve
python ../.claude/scripts/validate_endpoints.py

# Full combined validation
python ../.claude/scripts/validate_app.py
```

Fix all reported errors before connecting — silent failures are the most common time sink when onboarding a new backend.

---

## Key Configuration Files

| File | Purpose |
|------|---------|
| `widgets.json` | Defines every widget: type, endpoint, params, columnDefs, grid size. Must be a **JSON object** keyed by widget ID — not an array. |
| `apps.json` | Tab layout, widget positions on the 40-column grid, parameter sync groups. Must be a **JSON array** — even for a single app. |
| `data_loader.py` | Pre-loads all CSV data at startup into memory. Core data never hits disk on a live request. |
| `plotly_templates.py` | Dark/light Plotly theme helpers shared across all chart routes. |
| `AGENTS.md` | Quick-reference sheet of OpenBB schema rules and backend architecture patterns, written for AI coding agents. |

---

## AI Agent Skills (`.claude/skills/`)

These skills are the development toolkit used to build this app. Any AI agent (Claude, Gemini, Cursor, etc.) pointed at this repo can invoke them to build, extend, or validate OpenBB backends from scratch.

### `openbb-app-builder`
A 7-phase pipeline for building production-ready OpenBB backends:

| Phase | Description |
|-------|-------------|
| 1 — Interview | Requirements gathering, reference app analysis (`APP-INTERVIEW.md`) |
| 2 — Widget Metadata | Define widget types, params, column definitions (`WIDGET-METADATA.md`) |
| 3 — Dashboard Layout | 40-column grid layout design (`DASHBOARD-LAYOUT.md`) |
| 4 — Plan | Step-by-step implementation plan (`APP-PLANNER.md`) |
| 5 — Build | Generate all files: `main.py`, `widgets.json`, `apps.json`, routes |
| 6 — Validate | Run validation scripts against schema rules (`VALIDATE.md`) |
| 7 — Browser Test | Live test against OpenBB Workspace (`APP-TESTER.md`) |

**Usage with Claude (Cowork / Claude Code):**
```
Build an OpenBB app for [your use case]
```
Claude reads the skill files and executes the full pipeline automatically.

### `api-discovery`
Explores APIs and data sources before building. Identifies data shapes, latency characteristics, and the right widget type for each endpoint. Run this before `openbb-app-builder` when integrating a new data source.

### `odp-backend`
Covers ODP credential management (key hierarchy, `user_settings.json`), data pre-loading patterns, and OpenBB AI Copilot agent implementation.

### `improve-openbb-skill`
Submits fixes and improvements to the upstream [backends-for-openbb](https://github.com/OpenBB-finance/backends-for-openbb) repo via automated PR.

---

## Cached OpenBB Documentation (`.claude/docs/`)

Local documentation cache — agents read these before fetching from the web. Eliminates network dependency during development.

| File | Topic |
|------|-------|
| `INDEX.md` | Navigation guide — which doc to open for which question |
| `workspace-fundamentals.md` | Workspace architecture, widgets, dashboards |
| `odp-data-integration.md` | Custom backend, `widgets.json` schema, `columnsDefs` |
| `api-keys.md` | API key management and ODP credential hierarchy |
| `openbb-ai-sdk.md` | OpenBB AI SDK: agents, QueryRequest, helpers |
| `agent-patterns.md` | 10 agent implementation patterns |
| `sse-protocol-models.md` | SSE protocol and Pydantic models |
| `apps-configuration.md` | `apps.json`: tabs, groups, layout |
| `mcp-server.md` | MCP server integration |
| `testing.md` | Agent testing: CopilotResponse, payloads |

---

## Critical OpenBB Schema Rules

Violations cause **silent failures** — widgets simply don't appear or sync. These were learned through iteration against live OpenBB Workspace.

| Rule | Detail |
|------|--------|
| `widgets.json` is an **object** | `{"widget_id": {...}}` — NOT an array |
| `apps.json` is an **array** | `[{...}]` — even for a single app |
| Required app fields | `name`, `description`, `allowCustomization`, `tabs`, `groups`, `prompts` |
| Group names | Must be `"Group 1"`, `"Group 2"`, etc. — custom names fail silently |
| Group objects | Must include the `"name"` field explicitly |
| Layout item IDs | Use `"i"` field (not `"id"`) to reference widget IDs |
| Layout positions | Flat `x`, `y`, `w`, `h` — not nested inside `gridData` |
| `formatterFn` values | `"int"`, `"none"`, `"percent"` — `"currency"` is **not valid** |
| Plotly chart titles | **Omit** — the widget frame provides its own title |
| `advanced_charting` | Does NOT support parameter groups; use `chart` (Plotly) for interactivity |

---

## Connecting Live Data

The app currently runs on synthetic SFS-style data. To connect live systems:

- **Markets / Deals:** Replace CSV reads in `routes/markets.py` with your internal deal API or database
- **Portfolio:** Swap `data/hre_*.csv` with live Salesforce / internal system queries in `routes/portfolio.py`
- **SharePoint:** Update `routes/sharepoint.py` with MS Graph API credentials and your tenant folder paths
- **Forms / CRM:** Connect `routes/forms.py` write endpoints to your Salesforce or CRM API

**API key management** follows the ODP credential hierarchy:
1. `~/.openbb_platform/user_settings.json` (credentials block) — highest priority
2. Environment variable
3. `.env` file — never overrides ODP keys

---

## References

| Resource | Link |
|----------|------|
| OpenBB Workspace | https://pro.openbb.co |
| OpenBB Docs | https://docs.openbb.co/workspace |
| ODP (Custom Backends) | https://docs.openbb.co/odp |
| API Key Management | https://docs.openbb.co/odp/desktop/api-keys |
| OpenBB Examples | https://github.com/OpenBB-finance/awesome-openbb |
| Backends for OpenBB | https://github.com/OpenBB-finance/backends-for-openbb |
| OpenBB LLM Docs | https://docs.openbb.co/workspace/llms-full.txt |
