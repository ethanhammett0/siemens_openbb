# ODP & Custom Backend Data Integration

Sources: https://docs.openbb.co/odp, https://docs.openbb.co/workspace/developers/data-integration (fetched 2026-04-10)

## What is ODP?

Open Data Platform — the open-source toolset for integrating proprietary, licensed, and public data sources into downstream applications (AI copilots, research dashboards).

**"Connect once, consume everywhere"** — consolidates data across:
- Python environments (quantitative analysis)
- OpenBB Workspace + Excel (analyst workflows)
- MCP servers (AI agents)
- REST APIs (external applications)

## Three ODP Interfaces

1. **ODP Desktop** — standalone app for managing Python environments and backend servers
2. **ODP Python** — PyPI packages for building SDKs, REST APIs, MCP servers
3. **ODP CLI** — command-line wrapper around installed ODP Python packages

All three share identical infrastructure underneath.

## Custom Backend Architecture

A custom backend is a FastAPI server returning data in OpenBB-compatible formats, configured via `widgets.json`.

### Required Endpoints

```
GET  /               → API info / health check
GET  /widgets.json   → widget definitions (metadata, params, endpoints)
GET  /apps.json      → app layouts (tabs, groups, widget placement)
POST /<endpoint>     → actual data endpoints matching widget definitions
```

### Server Setup Pattern

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://pro.openbb.co"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Authentication

Use custom headers (not URL params):
- Set API keys as env vars via `.env`
- Extract from request headers: `X-API-KEY`
- Validate before processing

### Running

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 7779
```

Integration: Dashboard → "Add data" → enter base URL → credentials auto-configured via UI.

## widgets.json Structure

```json
{
  "widget_endpoint_id": {
    "name": "Display Name",
    "description": "What this widget shows",
    "category": "Category",
    "type": "table|chart|markdown|metric|file_viewer",
    "endpoint": "endpoint_path",
    "gridData": { "w": 20, "h": 9 },
    "params": {
      "param_name": {
        "type": "text|date|dropdown|ticker",
        "label": "Display Label",
        "description": "Help text",
        "value": "default_value",
        "options": ["opt1", "opt2"]
      }
    },
    "columnsDefs": [
      {
        "field": "column_name",
        "headerName": "Display Name",
        "formatterFn": "greenRed|int|float|percent|currency",
        "pinned": "left|right",
        "cellOnClick": { "type": "navigate", "url": "..." }
      }
    ]
  }
}
```

### Widget Display Types

| Type | Use Case | Data Format |
|------|----------|-------------|
| `table` | Tabular data (AgGrid) | JSON array of objects |
| `chart` | Plotly charts | Plotly JSON spec |
| `markdown` | Formatted text | Markdown string |
| `metric` | Single KPI values | `{ value, label, delta }` |
| `file_viewer` | PDFs, images | File URL or base64 |

### Parameter Types

| Type | Renders As |
|------|-----------|
| `text` | Text input |
| `date` | Date picker |
| `dropdown` | Select from options list |
| `ticker` | Ticker symbol input |
| `number` | Numeric input |
| `boolean` | Toggle |
| `endpoint` | Dynamic options from API |
| `tabs` | Tab-style selector |

### columnsDefs Formatters

| formatterFn | Effect |
|-------------|--------|
| `greenRed` | Color positive green, negative red |
| `int` | Integer formatting |
| `float` | Float with decimals |
| `percent` | Percentage with % symbol |
| `currency` | Dollar sign formatting |

Use `pinned: "left"` to freeze columns. Use `cellOnClick` for interactive cells.
