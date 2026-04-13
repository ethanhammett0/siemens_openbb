# Siemens Financial Services — OpenBB Workspace App

Custom backend and AI agent skills for building OpenBB Workspace dashboards tailored to Siemens Financial Services (SFS) structured finance workflows.

---

## Repository Structure

```
siemens_openbb/
├── structured_finance_app/     # FastAPI backend — runs at port 7800
│   ├── main.py                 # App entry point, CORS config, router mounts
│   ├── widgets.json            # Widget definitions (object keyed by widget ID)
│   ├── apps.json               # Dashboard layout config (array)
│   ├── data_loader.py          # Data pre-loading at startup
│   ├── plotly_templates.py     # Shared Plotly theme helpers
│   ├── requirements.txt        # Python dependencies
│   ├── run.bat                 # Windows launcher
│   ├── routes/
│   │   ├── markets.py          # Deal flow distribution, scatter, radar, raincloud
│   │   ├── portfolio.py        # Healthcare RE portfolio — assets, tenants, maturities
│   │   ├── forms.py            # Salesforce-style deal entry, tranche & account tracking
│   │   └── sharepoint.py       # Document browser / PDF viewer
│   ├── data/                   # Synthetic HRE portfolio CSVs
│   └── forms_data/             # SFS pipeline, tranche, account, and RE asset CSVs
│
└── .claude_skills/             # AI agent skills for OpenBB development
    ├── openbb-app-builder/     # Full 7-phase build pipeline
    ├── api-discovery/          # API/data exploration before building
    ├── odp-backend/            # ODP credentials, data pre-loading, copilot agents
    └── improve-openbb-skill/   # Submit fixes upstream to OpenBB-finance
```

---

## Structured Finance App

A unified FastAPI backend combining four dashboards for SFS internal use:

| Dashboard | Description |
|-----------|-------------|
| **Markets Monitor** | Deal flow distribution by interval/metric, scatter analysis (LTV vs spread), raincloud density, radar benchmarking |
| **Healthcare Portfolio Manager** | Asset-level portfolio grid with row grouping, cash flow history, allocation charts |
| **Salesforce App** | Deal entry, tranche tracking, real estate assets, and account lookup backed by SFS pipeline data |
| **SharePoint Viewer** | Browse and view deal documents and PDFs from SharePoint folders |

### Quick Start

**Prerequisites:** Python 3.10+

```bash
cd structured_finance_app
pip install -r requirements.txt
uvicorn main:app --reload --port 7800
```

**Connect to OpenBB Workspace:**

1. Open [pro.openbb.co](https://pro.openbb.co)
2. Settings → Data Connectors → Add Backend
3. Enter: `http://localhost:7800`
4. Validate — all dashboards will appear in your app gallery

### Key Files

| File | Purpose |
|------|---------|
| `widgets.json` | Defines every widget: type, endpoint, params, column defs, grid size |
| `apps.json` | Tab layout, widget positions (40-col grid), parameter sync groups |
| `data_loader.py` | Pre-loads CSV data at startup — no per-request I/O for core data |
| `plotly_templates.py` | Dark/light theme helpers for Plotly charts |

---

## AI Agent Skills

The `.claude_skills/` directory contains a complete development toolkit for building OpenBB Workspace apps with AI agents (Claude, Gemini, etc.). These are the same skills used to build this app.

### openbb-app-builder

A 7-phase pipeline for building production-ready OpenBB backends from scratch:

| Phase | Description |
|-------|-------------|
| 1 — Interview | Requirements gathering, reference app analysis |
| 2 — Widget Metadata | Define widget types, params, column defs |
| 3 — Dashboard Layout | 40-column grid layout design |
| 4 — Plan | Step-by-step implementation plan |
| 5 — Build | Generate all files (main.py, widgets.json, apps.json) |
| 6 — Validate | Run validation scripts against schema rules |
| 7 — Browser Test | Live test against OpenBB Workspace |

Reference docs in `openbb-app-builder/references/`:
- `WIDGET-METADATA.md` — column defs, formatterFn, renderFn, param types
- `PARAMETER-OPTIMIZATION.md` — parameter architecture, multi-select, ontology
- `DASHBOARD-LAYOUT.md` — grid positioning, tab design, layout templates
- `VALIDATE.md` — pre-deployment checklist and common error fixes
- `APP-INTERVIEW.md` — structured requirements interview process

### api-discovery

Explores APIs and data sources before building — identifies data shapes, latency characteristics, and optimal widget types. Use before `openbb-app-builder` when integrating a new data source.

### odp-backend

Patterns for ODP credential management, data pre-loading at startup, and OpenBB AI Copilot agent implementation.

### Usage with Claude (Cowork / Claude Code)

Point Claude at this repo and invoke the skill:

```
Build an OpenBB app for [your use case]
```

Claude will read the skill files and run the full 7-phase pipeline automatically.

---

## OpenBB Schema — Critical Rules

These rules are validated by OpenBB Workspace at runtime. Violations cause silent failures.

| Rule | Detail |
|------|--------|
| `widgets.json` is an **object** | `{"widget_id": {...}}` — NOT an array |
| `apps.json` is an **array** | `[{...}]` — even for a single app |
| Required app fields | `name`, `description`, `allowCustomization`, `tabs`, `groups`, `prompts` |
| Group names | Must be `"Group 1"`, `"Group 2"`, etc. — custom names fail silently |
| Layout item IDs | Use `"i"` field (not `"id"`) to reference widget IDs |
| `formatterFn` values | `"int"`, `"none"`, `"percent"` — `"currency"` is invalid |
| Plotly chart titles | Omit — the widget frame provides its own title |
| `advanced_charting` | Does NOT support parameter groups; use `chart` (Plotly) for interactivity |

---

## Data Sources

The current app runs on synthetic SFS-style data. To connect live data:

- **Markets / Deals**: Replace CSV reads in `routes/markets.py` with your internal API or database
- **Portfolio**: Swap `data/hre_*.csv` with live Salesforce / internal system queries
- **SharePoint**: Update `routes/sharepoint.py` with MS Graph API credentials
- **Forms**: Connect `routes/forms.py` to your Salesforce or CRM write endpoints

API key management follows the ODP credential hierarchy:
1. `~/.openbb_platform/user_settings.json` (credentials block)
2. Environment variable
3. `.env` file (never overrides ODP keys)

---

## References

| Resource | Link |
|----------|------|
| OpenBB Workspace | https://pro.openbb.co |
| OpenBB Docs | https://docs.openbb.co/workspace |
| ODP (Custom Backends) | https://docs.openbb.co/odp |
| OpenBB Examples | https://github.com/OpenBB-finance/awesome-openbb |
| Backends for OpenBB | https://github.com/OpenBB-finance/backends-for-openbb |
| OpenBB LLM Docs | https://docs.openbb.co/workspace/llms-full.txt |
