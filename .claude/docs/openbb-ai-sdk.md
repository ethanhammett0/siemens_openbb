# OpenBB AI SDK (openbb-ai) — Complete Reference

Source: https://github.com/OpenBB-finance/openbb-ai — cloned and read 2026-04-10

## Installation

```bash
pip install openbb-ai
```

## Exports

```python
from openbb_ai import (
    QueryRequest,      # Main request model — THE entrypoint
    Widget,            # Widget definition model
    WidgetRequest,     # Widget data retrieval request
    message_chunk,     # Stream text to frontend
    reasoning_step,    # Status updates / thought steps
    get_widget_data,   # Request widget data from Workspace
    cite,              # Create a single citation
    citations,         # Stream citations to frontend
    table,             # Inline table artifact
    chart,             # Inline chart artifact (line/bar/scatter/pie/donut)
)
```

## Required Endpoints

Every agent needs exactly two endpoints:

### 1. `/agents.json` — Agent Discovery

```python
@app.get("/agents.json")
async def agents_json():
    return JSONResponse(content={
        "my_agent_id": {
            "name": "My Agent",
            "description": "What it does",
            "image": "https://example.com/logo.png",
            "endpoints": {"query": "/v1/query"},
            "features": {
                "streaming": True,                   # MUST be True
                "widget-dashboard-select": True,      # access primary widgets
                "widget-dashboard-search": True,      # access secondary widgets
                # Custom toggle features:
                "deep-research": {
                    "label": "Deep Research",
                    "default": False,
                    "description": "Enable deep research mode",
                },
                # MCP tool support:
                "mcp-tools": True,
            },
        }
    })
```

### 2. `/query` (POST) — Main Agent Logic

```python
@app.post("/v1/query")
async def query(request: QueryRequest) -> EventSourceResponse:
    async def execution_loop():
        yield reasoning_step("Processing...").model_dump()
        yield message_chunk("Hello!").model_dump()
    return EventSourceResponse(content=execution_loop(), media_type="text/event-stream")
```

## QueryRequest — The Core Model

**Stateless architecture**: every request carries full state. Fields:

| Field | Type | Description |
|---|---|---|
| `messages` | `list[LlmClientMessage \| LlmClientFunctionCallResultMessage]` | Full conversation history |
| `widgets` | `WidgetCollection \| None` | Primary, secondary, extra widget groups |
| `context` | `list[RawContext] \| None` | Additional context (auto-includes yielded artifacts) |
| `urls` | `list[str] \| None` | Up to 4 URLs to retrieve as context |
| `api_keys` | `UserAPIKeys \| None` | Custom API keys for request |
| `timezone` | `str` | Default "UTC" |
| `workspace_state` | `WorkspaceState \| None` | Dashboard info, tabs, agents list |
| `workspace_options` | `list[str] \| None` | Enabled custom features (e.g. `["deep-research"]`) |
| `tools` | `list[AgentTool] \| None` | Available MCP tools |

## Helper Functions

### message_chunk(text: str) → MessageChunkSSE
Stream text back to Workspace. Yield `.model_dump()`.

### reasoning_step(message, event_type="INFO", details=None) → StatusUpdateSSE
Show status/thought steps. `event_type`: "INFO", "WARNING", "ERROR".
`details` can include key-value pairs shown in the UI.

### get_widget_data(widget_requests: list[WidgetRequest]) → FunctionCallSSE
**Remote function call** — triggers a two-phase interaction:
1. Yield this → Workspace fetches the data → closes connection
2. Workspace sends a follow-up POST to `/query` with data in `messages[-1]` (role="tool")

```python
# Phase 1: Request data
for widget in request.widgets.primary:
    widget_requests.append(WidgetRequest(
        widget=widget,
        input_arguments={p.name: p.current_value for p in widget.params},
    ))
yield get_widget_data(widget_requests).model_dump()
# Connection closes. Phase 2 comes as a new request with role="tool" last message.
```

### cite(widget, input_arguments, extra_details=None) → Citation
Create citation for a widget. Use with `citations()`.

### citations(citations_list: list[Citation]) → CitationCollectionSSE
Stream citations to Workspace. Typically yielded at end of response.

### table(data: list[dict], name=None, description=None) → MessageArtifactSSE
Inline table artifact in chat. Each dict = one row.

### chart(type, data, x_key, y_keys, angle_key, callout_label_key, name, description) → MessageArtifactSSE
Inline chart in chat. Types:
- **line/bar/scatter**: requires `x_key` + `y_keys`
- **pie/donut**: requires `angle_key` + `callout_label_key`

## Widget Data Flow Architecture

```
OpenBB Workspace                    Your Agent Backend
       │                                │
       │  1. POST /query                │
       │  { messages, widgets }         │
       │───────────────────────────────>│
       │                                │
       │  2. FunctionCallSSE            │
       │  (get_widget_data)             │
       │<───────────────────────────────│
       │  (connection closes)           │
       │                                │
       │  3. POST /query                │
       │  { messages: [                 │
       │    ...original,                │
       │    function_call,              │
       │    function_call_result        │  ← role="tool", data=[...]
       │  ]}                            │
       │───────────────────────────────>│
       │                                │
       │  4. SSE stream                 │
       │  (text, reasoning, citations)  │
       │<───────────────────────────────│
```

## Available Function Calls (FunctionCallSSEData.function)

```python
Literal[
    "get_widget_data",            # Retrieve widget data
    "get_extra_widget_data",      # Retrieve extra widget data
    "get_params_options",         # Get parameter options
    "add_widget_to_dashboard",    # Add widget to dashboard
    "add_generative_widget",      # Add AI-generated widget
    "update_widget_in_dashboard", # Update existing widget
    "assign_tasks_to_agents",     # Delegate to other agents
    "execute_agent_tool",         # Execute MCP tool
    "manage_navigation_bar",      # Manage navigation
    "get_skill_content",          # Load dynamic skill
]
```

## Widget Model Fields

```python
class Widget(BaseModel):
    uuid: UUID              # Auto-generated if not provided
    origin: str             # Source identifier (e.g. "OpenBB API")
    widget_id: str          # Endpoint ID
    name: str               # Display name
    description: str        # What it shows
    params: list[WidgetParam]  # Parameter definitions
    source: str | None      # Data provider source
    category: str | None
    sub_category: str | None
    columns: list[str] | None  # Column names for table widgets
    metadata: dict
```

## WidgetParam Types

```python
Literal["string", "text", "number", "integer", "boolean", "date", "ticker", "endpoint", "tabs"]
```

Key fields: `name`, `type`, `description`, `default_value`, `current_value`, `multi_select`, `options`, `get_options`.

## Data Format Types

The SDK handles multiple data formats:
- `RawObjectDataFormat` — JSON objects, parse_as: table/chart/text/html/snowflake_query/snowflake_python
- `PdfDataFormat` — PDF files
- `ImageDataFormat` — jpg/jpeg/png
- `SpreadsheetDataFormat` — xlsx/xls/csv
- `PlaintextDataFormat` — txt/md
- `DocxDataFormat` — Word documents

## Artifact Types

```python
ArtifactTypes = Literal["text", "table", "chart", "snowflake_query", "snowflake_python", "html"]
```
