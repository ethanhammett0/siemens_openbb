# OpenBB Workspace — Agent Rules & Reference

> This file is read by AI coding agents (Antigravity, Claude, Gemini, etc.) to guide OpenBB Workspace backend and widget development in this repository. All rules here are authoritative for any work done in this codebase.

---

## 🗂️ Repository Layout

```
openbb_widgets/
├── AGENTS.md                  # ← You are here. Agent rules & OpenBB reference.
├── .agents/skills/            # Agent-agnostic extended skill pipeline (all agents)
│   ├── openbb-app-builder/    #   Full 7-phase build pipeline + reference docs
│   └── improve-openbb-skill/  #   Submit upstream fixes to the skill repo
├── .claude/skills/            # Claude Code mirror of the above (Claude-specific)
├── markets_monitor/           # Market monitoring app (port 7779)
├── portfolio_review/          # Private credit portfolio app (port 8015)
├── form_widgets/              # Form-based widgets app
├── sharepoint_widget/         # SharePoint document viewer
├── deal_comparison/           # Deal comparison app
├── sponsor_dashboard/         # Sponsor dashboard app
├── dummy_data/                # Shared synthetic data generation (dummy_data.py)
└── openbb_documentation/      # OpenBB doc references
```

> **Note for agents**: Extended skill pipeline lives in `.agents/skills/openbb-app-builder/` (agent-agnostic) and is mirrored in `.claude/skills/openbb-app-builder/` for Claude Code. The rules in this `AGENTS.md` are the authoritative quick reference; fall back to the skill files for deep-dive pipeline detail.

---

## 🚦 Critical Rules — Read These First

These rules have been learned through trial and error against real OpenBB Workspace validation. Violations cause silent failures or hard validation errors.

| # | Rule | Detail |
|---|------|--------|
| 1 | **`widgets.json` must be a JSON object** | `{"widget_id": {...}}` — NOT an array `[...]` |
| 2 | **`apps.json` must be an array** | `[{...}]` — despite what some docs say |
| 3 | **Each app in `apps.json` must have**: `name`, `description`, `allowCustomization`, `tabs`, `groups`, `prompts` | Missing any causes browser validation failure |
| 4 | **Group names MUST be `"Group 1"`, `"Group 2"`, etc.** | Custom names like `"symbol-group"` fail silently; widgets won't sync |
| 5 | **Group objects MUST include the `name` field** | `{"name": "Group 1", "type": "param", ...}` |
| 6 | **Layout items use `i` for widget ID**, not `id` | `{"i": "widget_id", "x": 0, "y": 0, "w": 20, "h": 12}` |
| 7 | **Layout positions are flat** (`x`, `y`, `w`, `h` directly) | NOT nested inside `gridData` |
| 8 | **`"currency"` is NOT a valid `formatterFn`** | Use `"none"` for currency/decimal display instead |
| 9 | **Plotly charts must NOT include a title** | The widget frame already provides its own title |
| 10 | **`advanced_charting` (TradingView) does NOT support parameter groups** | Use `chart` (Plotly) if you need click-to-update behavior |

---

## 🏗️ Backend Architecture

Every OpenBB backend is a **FastAPI app** with CORS enabled for the three required origins.

```python
from fastapi import FastAPI, Query
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import json
from pathlib import Path

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://pro.openbb.co",
        "https://pro.openbb.dev",
        "http://localhost:1420"
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Required endpoints
@app.get("/widgets.json")
def get_widgets():
    return {  # MUST be dict with widget IDs as keys
        "my_widget": {
            "name": "My Widget",
            "type": "table",
            "endpoint": "my_endpoint"
        }
    }

@app.get("/apps.json")
def get_apps():
    APPS_FILE = Path(__file__).parent / "apps.json"
    with open(APPS_FILE) as f:
        return json.load(f)  # MUST be an array: [{...}]
```

### Run Locally
```bash
uvicorn main:app --reload --port 7779
```

### Add to OpenBB Workspace
Settings → Data Connectors → Add Backend: `http://localhost:7779`

### Refreshing After Changes
- **Widget config changes** (widgets.json / apps.json): Right-click widget → "Refresh backend"
- **Python code changes**: Restart uvicorn
- **Major structural changes**: Open a fresh app instance from the gallery

---

## 📦 File Structure for Each App

```
{app-name}/
├── main.py            # FastAPI application
├── widgets.json       # Widget configurations (object, NOT array)
├── apps.json          # Dashboard layout (array, NOT object)
├── requirements.txt   # Python dependencies
└── .env.example       # Environment template
```

---

## 🧩 Widget Types

| Type | Use Case | Grouping Support |
|------|----------|-----------------|
| `table` | Tabular data with sorting/filtering | ✅ Yes |
| `chart` | Plotly visualizations | ✅ Yes |
| `metric` | KPI values with label + delta | ✅ Yes |
| `markdown` | Formatted text content | ✅ Yes |
| `newsfeed` | Article lists | ✅ Yes |
| `html` | Custom HTML (no JS) | ✅ Yes |
| `pdf` | PDF viewer | ✅ Yes |
| `live_grid` | Real-time table | ✅ Yes |
| `omni` | AI responses / mixed content | ✅ Yes |
| `advanced_charting` | TradingView charts | ❌ **NO** grouping |

---

## 📐 `widgets.json` Structure

```json
{
  "widget_id": {
    "name": "Display Name",
    "description": "Widget description",
    "category": "Category Name",
    "type": "table",
    "endpoint": "endpoint_path",
    "gridData": {"w": 20, "h": 12},
    "params": [
      {
        "paramName": "symbol",
        "type": "endpoint",
        "label": "Symbol",
        "optionsEndpoint": "/symbols",
        "value": "AAPL"
      }
    ],
    "columns": [
      {
        "field": "symbol",
        "headerName": "Symbol",
        "cellDataType": "text",
        "pinned": "left"
      },
      {
        "field": "price",
        "headerName": "Price",
        "cellDataType": "number",
        "formatterFn": "int"
      },
      {
        "field": "change_pct",
        "headerName": "Change %",
        "cellDataType": "number",
        "formatterFn": "percent",
        "renderFn": "greenRed"
      }
    ]
  }
}
```

### Parameter Types

| Type | Description | Example |
|------|-------------|---------|
| `text` | Text input (or static dropdown with `options`) | Search query, interval |
| `number` | Numeric input | Limit, count |
| `boolean` | Toggle | Include extended hours |
| `date` | Date picker | Start date |
| `endpoint` | Dynamic dropdown from API | Symbol selector |

**Date modifiers**: `$currentDate`, `$currentDate-1d`, `$currentDate-1w`, `$currentDate-1M`, `$currentDate-1y`

**Static dropdown** (use `type: "text"` + `options`):
```json
{
  "paramName": "interval",
  "type": "text",
  "label": "Interval",
  "value": "1d",
  "options": [
    {"label": "1 Day", "value": "1d"},
    {"label": "1 Week", "value": "1w"}
  ]
}
```

**Dynamic dropdown** (use `type: "endpoint"` + `optionsEndpoint`):
```json
{
  "paramName": "symbol",
  "type": "endpoint",
  "label": "Symbol",
  "optionsEndpoint": "/symbols",
  "multiSelect": false
}
```

### Column `formatterFn` Valid Values
- `int` — Integer formatting
- `none` — No formatting (use for currency/decimals)
- `percent` — Percentage formatting
- `normalized` — Normalize to scale
- `normalizedPercent` — Normalized percentage
- `dateToYear` — Extract year from date

### Column `renderFn` Values
- `greenRed` — Positive=green, Negative=red
- `titleCase` — Capitalize words
- `hoverCard` — Show markdown on hover
- `cellOnClick` — Click-to-update (watchlist pattern)
- `columnColor` — Conditional coloring
- `showCellChange` — Animate value changes

---

## 📊 `apps.json` Structure

> **CRITICAL**: `apps.json` must be a **JSON array** `[{...}]`, even if there is only one app.

```json
[
  {
    "name": "My Dashboard",
    "description": "Dashboard description",
    "img": "",
    "img_dark": "",
    "img_light": "",
    "allowCustomization": true,
    "prompts": [],
    "tabs": {
      "": {
        "id": "",
        "name": "",
        "layout": [
          {
            "i": "widget_id",
            "x": 0,
            "y": 0,
            "w": 20,
            "h": 12,
            "groups": ["Group 1"]
          }
        ]
      }
    },
    "groups": [
      {
        "name": "Group 1",
        "type": "param",
        "paramName": "symbol",
        "defaultValue": "AAPL"
      }
    ]
  }
]
```

### Required App Fields

| Field | Type | Notes |
|-------|------|-------|
| `name` | string | Display name |
| `description` | string | What the app does |
| `img`, `img_dark`, `img_light` | string | Image URLs (can be `""`) |
| `allowCustomization` | boolean | Usually `true` |
| `prompts` | array | Can be `[]` |
| `tabs` | object | Tab configurations |
| `groups` | array | Parameter sync groups (can be `[]`) |

### Tab Configurations

**Single tab (no tab bar)** — use empty string for id/name:
```json
"tabs": {
  "": {
    "id": "",
    "name": "",
    "layout": [...]
  }
}
```

**Multiple named tabs**:
```json
"tabs": {
  "overview": {"id": "overview", "name": "Overview", "layout": [...]},
  "details":  {"id": "details",  "name": "Details",  "layout": [...]}
}
```

### Layout Item Fields

| Field | Description |
|-------|-------------|
| `i` | Widget ID — must match key in `widgets.json` |
| `x` | X position (0–39) |
| `y` | Y position |
| `w` | Width (10–40) |
| `h` | Height (4+) |
| `groups` | e.g. `["Group 1"]` to sync parameters |
| `state` | (Optional) Pre-configure widget display |

### Layout Item `state` (Pre-configure Display)

```json
{
  "i": "my_table",
  "x": 0, "y": 0, "w": 40, "h": 14,
  "state": {
    "chartView": {
      "enabled": true,
      "chartType": "line"
    },
    "chartModel": {
      "modelType": "range",
      "chartType": "line",
      "cellRange": {
        "columns": ["date", "value1", "value2"]
      },
      "suppressChartRanges": true
    },
    "columnState": {
      "default": {
        "rowGroup": {"groupColIds": ["category"]},
        "columnVisibility": {"hiddenColIds": []},
        "columnOrder": {"orderedColIds": ["col1", "col2", "col3"]}
      }
    }
  },
  "groups": ["Group 1"]
}
```

---

## 🔗 Interactive Watchlist Pattern (cellOnClick + groupBy)

Make table cells clickable to update other widgets in the same group:

**In `widgets.json` column definition:**
```json
{
  "field": "symbol",
  "headerName": "Symbol",
  "cellDataType": "text",
  "pinned": "left",
  "renderFn": "cellOnClick",
  "renderFnParams": {
    "actionType": "groupBy",
    "groupByParamName": "symbol"
  }
}
```

**Requirements:**
1. Both table and target widget must be in the same group: `"groups": ["Group 1"]`
2. Target widget MUST be `type: "chart"` (Plotly) — NOT `advanced_charting`
3. Both widgets need matching `paramName` with `type: "endpoint"`
4. Group names MUST follow the `"Group N"` pattern

---

## 🗺️ Grid Layout System

- OpenBB Workspace uses a **40-column grid**
- Minimum width: 10 columns
- Maximum width: 40 columns (full width)
- Minimum height: 4 rows

### Height Guidelines

| Widget Type | Recommended Height |
|-------------|-------------------|
| `metric` | 4–6 |
| `table` (small) | 8–12 |
| `table` (medium) | 12–15 |
| `chart` | 12–15 |
| `newsfeed` | 12–15 |
| `markdown` | 8–12 |

### Common Layout Templates

**Overview Dashboard:**
```
+-------------------------------------------+  row 0
|              [Metrics Bar]   w=40, h=4    |
+--------------------+----------------------+  row 4
|  [Main Chart]      |    [Data Table]      |
|  w=20, h=15        |    w=20, h=15        |
+--------------------+----------------------+  row 19
```

**Watchlist + Chart:**
```
+-------------------------------------------+  row 0
|     [Watchlist] w=40, h=8  (cellOnClick)  |
+-------------------------------------------+  row 8
|     [Price Chart] w=40, h=15  (Plotly!)   |
+-------------------------------------------+  row 23
```

---

## 🐍 Widget Endpoint Patterns

### Table Endpoint
```python
@app.get("/stock_prices")
def stock_prices(symbol: str = Query("AAPL")):
    return [
        {"symbol": "AAPL", "price": 150.25, "change": 2.5},
        {"symbol": "GOOGL", "price": 140.50, "change": -1.2},
    ]
```

### Chart Endpoint (Plotly)
```python
import plotly.graph_objects as go

@app.get("/price_chart")
def price_chart(
    symbol: str = Query("AAPL"),
    theme: str = Query("dark"),
    raw: bool = Query(False)
):
    dates = ["2024-01-01", "2024-01-02", "2024-01-03"]
    prices = [150, 152, 148]

    if raw:
        return [{"date": d, "price": p} for d, p in zip(dates, prices)]

    fig = go.Figure()
    fig.add_trace(go.Scatter(x=dates, y=prices, mode="lines"))
    # NO title — the widget frame provides that
    fig.update_layout(
        template="plotly_dark" if theme == "dark" else "plotly_white"
    )
    return JSONResponse(content=json.loads(fig.to_json()))
```

### Metric Endpoint
```python
@app.get("/portfolio_metrics")
def portfolio_metrics():
    return [
        {"label": "Total Value", "value": "$125,430", "delta": "+5.2%"},
        {"label": "Daily P&L",   "value": "$2,340",   "delta": "+1.9%"},
    ]
```

### Markdown Endpoint
```python
@app.get("/analysis")
def analysis():
    return "## Market Summary\n\nKey insight here..."
```

### Newsfeed Endpoint
```python
@app.get("/news")
def news():
    return [
        {
            "title": "Market Rally Continues",
            "date": "2024-01-15",
            "author": "John Smith",
            "excerpt": "Stocks hit new highs...",
            "body": "Full article content here..."
        }
    ]
```

---

## ✅ Pre-Deployment Checklist

Before adding a backend to OpenBB Workspace, verify:

- [ ] `apps.json` is an **array** (starts with `[`)
- [ ] Each app has: `name`, `description`, `allowCustomization`, `tabs`, `groups`, `prompts`
- [ ] Each tab has: `id`, `name`, `layout`
- [ ] Layout items use `i` for widget ID (not `id`)
- [ ] Layout positions are flat: `x`, `y`, `w`, `h` (NOT inside `gridData`)
- [ ] All widget IDs in layout exist as keys in `widgets.json`
- [ ] `widgets.json` is an **object** (starts with `{`)
- [ ] `formatterFn` values are valid (no `"currency"` — use `"none"`)
- [ ] Group names follow `"Group N"` pattern
- [ ] CORS includes all three required origins
- [ ] Browser validation passes at `https://pro.openbb.co` (Settings → Data Connectors → Connect Backend → Test)

---

## 🐛 Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Unknown App: [name]: Required` | `apps.json` is object, not array | Change `{...}` to `[{...}]` |
| `[tabs]: Required` | Wrong tab structure | Each tab needs `id`, `name`, `layout` |
| `allowCustomization: Required` | Missing field | Add `"allowCustomization": true` |
| `groups: Required` | Missing field | Add `"groups": []` |
| `prompts: Required` | Missing field | Add `"prompts": []` |
| `Invalid formatterFn: currency` | Wrong value | Use `"none"` instead |
| `widgets.json must be object` | Array format used | Change `[{...}]` to `{"id": {...}}` |
| `Widget 'x' not found` | ID mismatch | Ensure `i` in layout matches key in widgets.json |
| `Invalid group name` | Custom group name | Use `"Group 1"`, `"Group 2"`, etc. |
| Widgets not syncing | Group name mismatch or `advanced_charting` | Use `"Group N"` names; use Plotly for interactive charts |
| CORS error | Missing origin | Add missing origin to FastAPI CORS middleware |

---

## 📚 External References

| Resource | URL |
|----------|-----|
| Latest OpenBB LLM docs | `https://docs.openbb.co/workspace/llms-full.txt` |
| OpenBB Workspace | `https://pro.openbb.co` |
| OpenBB Workspace (dev) | `https://pro.openbb.dev` |
| OpenBB examples repo | `https://github.com/OpenBB-finance/awesome-openbb` |
| Backends for OpenBB | `https://github.com/OpenBB-finance/backends-for-openbb` |

> **When documentation conflicts with browser validation errors, always trust the browser error.** Fetch `https://docs.openbb.co/workspace/llms-full.txt` for the most up-to-date schema.

---

## 🤖 Extended Agent Skills

The `.agents/skills/openbb-app-builder/` directory contains a full 7-phase build pipeline readable by any agent:
1. **Interview** — Requirements gathering → `references/APP-INTERVIEW.md`
2. **Widget Metadata** — Widget definitions → `references/WIDGET-METADATA.md`
3. **Dashboard Layout** — Visual layout design → `references/DASHBOARD-LAYOUT.md`
4. **Planner** — Step-by-step plan → `references/APP-PLANNER.md`
5. **Build** — Create all files → `references/OPENBB-APP.md`
6. **Validate** — Run validation scripts → `references/VALIDATE.md`
7. **Browser Test** — Verify against real OpenBB Workspace → `references/APP-TESTER.md`

A second skill, `.agents/skills/improve-openbb-skill/`, handles submitting documentation fixes upstream to `OpenBB-finance/backends-for-openbb`.

> Claude Code users: an identical mirror lives in `.claude/skills/openbb-app-builder/` for native Claude skill invocation.
