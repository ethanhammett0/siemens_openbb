# OpenBB App Implementation Reference

Core implementation knowledge for building OpenBB Workspace backends.

## Additional Documentation

For the latest documentation, fetch:
```
https://docs.openbb.co/workspace/llms-full.txt
```

## Open Source Examples

Curated examples at:
```
https://github.com/OpenBB-finance/awesome-openbb
```

---

## Core Requirements

Your backend must:

1. **Serve HTTP endpoints** returning JSON responses
2. **Enable CORS** for these origins:
   - `https://pro.openbb.co`
   - `https://pro.openbb.dev`
   - `http://localhost:1420`
3. **Implement required endpoints**:
   - `GET /widgets.json` — Return dict of widget configurations (NOT array)
   - `GET /apps.json` — Return dashboard configurations (MUST be array)
4. **Return proper Content-Type**: `application/json`

---

## Backend Architecture

```python
from fastapi import FastAPI, HTTPException, Query
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse

app = FastAPI()

# CORS - Required for OpenBB Workspace
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

@app.get("/")
def root():
    return {"message": "OpenBB Custom Backend"}

@app.get("/widgets.json")
def get_widgets():
    return {  # MUST be dict with widget IDs as keys
        "my_widget": {
            "name": "My Widget",
            "type": "table",
            "endpoint": "my_endpoint"
        }
    }
```

---

## Widget Types

### 1. Table Widget (type: "table")
Display tabular data with sorting, filtering, and chart conversion.

```python
@app.get("/stock_prices")
def stock_prices():
    return [
        {"symbol": "AAPL", "price": 150.25, "change": 2.5},
        {"symbol": "GOOGL", "price": 140.50, "change": -1.2},
    ]
```

### 2. Chart Widget (type: "chart")
Interactive Plotly charts with theme support.

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
    # NO title - widget provides it
    fig.update_layout(
        template="plotly_dark" if theme == "dark" else "plotly_white"
    )

    return JSONResponse(content=json.loads(fig.to_json()))
```

### 3. Metric Widget (type: "metric")
Display KPIs with labels, values, and deltas.

```python
@app.get("/portfolio_metrics")
def portfolio_metrics():
    return [
        {"label": "Total Value", "value": "$125,430", "delta": "+5.2%"},
        {"label": "Daily P&L", "value": "$2,340", "delta": "+1.9%"},
        {"label": "Win Rate", "value": "68%", "subvalue": "Last 30 days"},
    ]
```

### 4. Markdown Widget (type: "markdown")
Display formatted text content.

```python
@app.get("/analysis")
def analysis():
    return """
## Market Analysis

The market showed **strong momentum** today with:
- Tech sector up 2.3%
- Energy sector down 0.8%

### Recommendations
Buy: AAPL, MSFT
Sell: XOM
"""
```

### 5. Newsfeed Widget (type: "newsfeed")
Display articles with title, date, author, excerpt, and body.

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

## Widget Configuration

### widgets.json Structure

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
    ]
  }
}
```

### Parameter Types

| Type | Description | Example |
|------|-------------|---------|
| `text` | Text input | Search query |
| `number` | Number input | Limit, count |
| `boolean` | Toggle | Include extended |
| `date` | Date picker | Start date |
| `endpoint` | Dynamic dropdown | Symbol selector |

---

## apps.json Structure

**CRITICAL**: apps.json must be an **ARRAY** `[{...}]`, not an object. The backend must serve this via `GET /apps.json`.

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
          {"i": "widget_id", "x": 0, "y": 0, "w": 20, "h": 12, "groups": ["Group 1"]}
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

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name for the app |
| `description` | string | What the app does |
| `img`, `img_dark`, `img_light` | string | Image URLs (can be empty `""`) |
| `allowCustomization` | boolean | Whether users can modify layout |
| `prompts` | array | AI prompts (can be `[]`) |
| `tabs` | object | Tab configurations |
| `groups` | array | Parameter synchronization groups (can be `[]`) |

### Tab Configuration

**Single unnamed tab** (most common):
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
  "overview": {
    "id": "overview",
    "name": "Overview",
    "layout": [...]
  },
  "details": {
    "id": "details",
    "name": "Details",
    "layout": [...]
  }
}
```

### Layout Item Fields

| Field | Description |
|-------|-------------|
| `i` | Widget ID (must match key in widgets.json) |
| `x` | X position (0-39) |
| `y` | Y position |
| `w` | Width (10-40) |
| `h` | Height (4+) |
| `groups` | Array of group names, e.g. `["Group 1"]` |
| `state` | Optional: Pre-configure widget display |

### Layout Item State (Optional)

Pre-configure how widgets display using the `state` object:

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
        "rowGroup": {
          "groupColIds": ["category"]
        },
        "columnVisibility": {
          "hiddenColIds": []
        },
        "columnOrder": {
          "orderedColIds": ["col1", "col2", "col3"]
        }
      }
    }
  },
  "groups": ["Group 1"]
}
```

### Groups with Parameter Sync

**CRITICAL**: Group objects MUST include the `name` field, and names MUST follow "Group N" pattern.

```json
{
  "groups": [
    {
      "name": "Group 1",
      "type": "param",
      "paramName": "symbol",
      "defaultValue": "AAPL"
    }
  ]
}
```

---

## Column Definitions

For table widgets:

```json
{
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
      "field": "change",
      "headerName": "Change %",
      "cellDataType": "number",
      "formatterFn": "percent",
      "renderFn": "greenRed"
    }
  ]
}
```

### Valid formatterFn Values
- `int` — Integer formatting
- `none` — No formatting (use for currency)
- `percent` — Percentage formatting
- `normalized` — Normalize to scale
- `normalizedPercent` — Normalized percentage
- `dateToYear` — Extract year from date

**Note**: `"currency"` is NOT valid — use `"none"` instead.

---

## Best Practices

1. **No runButton by Default** — Only use `runButton: true` for heavy computation (>5 seconds)
2. **Reasonable Widget Heights** — Metrics: h=4-6, Tables: h=12-18, Charts: h=12-15
3. **Charts Without Titles** — Plotly charts should NOT include title (widget provides it)
4. **Support Raw Mode for Charts** — Add `raw` query parameter to return raw data for AI analysis
5. **Dict Format for widgets.json** — Return object with widget IDs as keys, NOT an array
6. **Array Format for apps.json** — Must be `[{...}]`, NOT an object `{...}`
7. **Group Names Pattern** — Use "Group 1", "Group 2" etc. — custom names fail silently

---

## Development Workflow

1. Start with template
2. Define data sources
3. Choose widget types
4. Configure parameters
5. Test locally: `uvicorn main:app --reload --port 7779`
6. Add to OpenBB: Settings > Data Connectors > Add Backend

### Refreshing Changes
- **Widget config changes**: Right-click → "Refresh backend"
- **Python code changes**: Restart uvicorn
- **Major changes**: Open fresh app instance from gallery

---

## Data Pre-loading Pattern

For backends with a known data universe, pre-load at startup and serve from memory. No per-request API calls for core data.

```python
import threading, asyncio
from contextlib import asynccontextmanager
from cachetools import TTLCache

# TTL caches for different freshness needs
_cache_5m = TTLCache(maxsize=256, ttl=300)      # Real-time: snapshots
_cache_1h = TTLCache(maxsize=512, ttl=3600)      # Moderate: aggregates
_cache_24h = TTLCache(maxsize=128, ttl=86400)    # Slow: reference data

@asynccontextmanager
async def lifespan(app: FastAPI):
    thread = threading.Thread(target=load_all_data, daemon=True)  # Non-blocking
    thread.start()
    task = asyncio.create_task(background_refresh(interval_minutes=15))
    yield
    task.cancel()

app = FastAPI(title="My Backend", lifespan=lifespan)
```

Background refresh should alternate incremental (re-fetch last 5 days) with full rebuilds (every ~1 hour). For user-requested tickers outside the pre-loaded universe, fetch on demand and add to cache with a thread lock.

---

## Copilot Agent Architecture

An ODP copilot agent serves two endpoints instead of `widgets.json`/`apps.json`:

| Endpoint | Method | Returns |
|----------|--------|---------|
| `/agents.json` | GET | Agent metadata + capabilities |
| `/query` | POST | SSE stream of response chunks |

### agents.json

```json
{
  "agent-id": {
    "name": "Display Name",
    "description": "What the agent does",
    "image": "https://...",
    "endpoints": {"query": "http://127.0.0.1:8015/query"},
    "features": {
      "streaming": true,
      "widget-dashboard-select": true,
      "widget-dashboard-search": true,
      "mcp-tools": true
    }
  }
}
```

Connect: Copilot settings → enter base URL. Workspace auto-discovers `/agents.json`.

### Query Endpoint

Install `openbb-ai`. Receives `QueryRequest` with: `messages` (history), `widgets` (primary/secondary), `workspace_state`, `tools` (MCP), `api_keys`.

SSE helpers: `message_chunk()`, `reasoning_step()`, `table()`, `chart()`, `citations()`, `get_widget_data()`. All yield `.model_dump()`.

Widget data is fetched statelessly: agent yields `get_widget_data()` → stream ends → frontend fetches → new POST with data in tool message.

### Pydantic AI Integration

```python
from openbb_pydantic_ai import OpenBBAIAdapter, OpenBBDeps
from pydantic_ai import Agent

agent = Agent('google-gla:gemini-3.1-pro-preview', deps_type=OpenBBDeps)

@app.post("/query")
async def query(request: Request):
    return await OpenBBAIAdapter.dispatch_request(request, agent=agent)
```

The adapter handles QueryRequest parsing, PDF extraction (docling), and SSE formatting.
