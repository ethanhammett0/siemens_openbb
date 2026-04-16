# SSE Protocol & Pydantic Models Reference

Source: openbb-ai/openbb_ai/models.py (972 lines, read 2026-04-10)

## SSE Event Types

All events extend `BaseSSE` and serialize via `.model_dump()`.

| Event Name | Class | Purpose |
|---|---|---|
| `copilotMessageChunk` | `MessageChunkSSE` | Stream text chunks to chat |
| `copilotMessageArtifact` | `MessageArtifactSSE` | Inline artifacts (table, chart, text, html) |
| `copilotFunctionCall` | `FunctionCallSSE` | Request Workspace to execute a function |
| `copilotStatusUpdate` | `StatusUpdateSSE` | Reasoning steps / status updates |
| `copilotCitationCollection` | `CitationCollectionSSE` | Citations linking to source widgets |

```python
SSE = MessageChunkSSE | MessageArtifactSSE | FunctionCallSSE | StatusUpdateSSE | CitationCollectionSSE
```

## Message Types

### LlmClientMessage (role: human | ai)
```python
class LlmClientMessage(BaseModel):
    role: RoleEnum             # "human" | "ai" | "tool"
    content: str | LlmClientFunctionCall
    agent_id: str | None       # e.g. "openbb-copilot" for orchestration
```

### LlmClientFunctionCallResultMessage (role: tool)
```python
class LlmClientFunctionCallResultMessage(BaseModel):
    role: RoleEnum = RoleEnum.tool
    function: str              # "get_widget_data", "execute_agent_tool", etc.
    input_arguments: dict      # What was requested
    data: list[DataContent | DataFileReferences | ClientFunctionCallError | ClientCommandResult]
    extra_state: dict
```

## Data Content Models

### SingleDataContent (inline data)
```python
class SingleDataContent(BaseModel):
    content: str               # Raw string, JSON string, or base64
    data_format: DataFormat    # How to parse (object, pdf, image, etc.)
    citable: bool = True
```

### SingleFileReference (file URL)
```python
class SingleFileReference(BaseModel):
    url: HttpUrl
    data_format: DataFormat
    citable: bool = True
```

### DataContent (container)
```python
class DataContent(BaseModel):
    items: list[SingleDataContent]
    extra_citations: list[Citation] = []
```

## Workspace State Models

```python
class WorkspaceState(BaseModel):
    action_history: list[str] | None
    agents: list[WorkspaceAgent] | None
    current_dashboard_uuid: UUID | None
    current_dashboard_info: DashboardInfo | None    # tabs, widgets
    current_page_context: str | None

class DashboardInfo(BaseModel):
    id: str
    name: str
    current_tab_id: str
    tabs: list[TabInfo] | None

class TabInfo(BaseModel):
    tab_id: str
    widgets: list[WidgetInfo] | None

class WidgetInfo(BaseModel):
    widget_uuid: str
    name: str
```

## Citation Models

```python
class Citation(BaseModel):
    id: UUID
    source_info: SourceInfo
    details: list[dict] | None
    quote_bounding_boxes: list[list[CitationHighlightBoundingBox]] | None

class SourceInfo(BaseModel):
    type: Literal["widget", "direct retrieval", "web", "artifact"]
    uuid: UUID | None
    origin: str | None
    widget_id: str | None
    name: str | None
    description: str | None
    metadata: dict          # typically {"input_args": {...}}
    citable: bool = True

class CitationHighlightBoundingBox(BaseModel):
    text: str
    page: int
    x0: float
    top: float
    x1: float
    bottom: float
```

## Chart Parameter Models

```python
# Line/Bar: xKey + yKey
class LineChartParameters(BaseModel):
    chartType: Literal["line"]
    xKey: str
    yKey: list[str]

class BarChartParameters(BaseModel):
    chartType: Literal["bar"]
    xKey: str
    yKey: list[str]

class ScatterChartParameters(BaseModel):
    chartType: Literal["scatter"]
    xKey: str       # numerical only
    yKey: list[str]  # numerical only

# Pie/Donut: angleKey + calloutLabelKey
class PieChartParameters(BaseModel):
    chartType: Literal["pie"]
    angleKey: str
    calloutLabelKey: str

class DonutChartParameters(BaseModel):
    chartType: Literal["donut"]
    angleKey: str
    calloutLabelKey: str
```

## Agent Tool Model (MCP)

```python
class AgentTool(BaseModel):
    server_id: str | None
    name: str
    url: str
    endpoint: str | None
    description: str | None
    input_schema: dict | None
    auth_token: str | None
```

## Key Enums & Literals

```python
ArtifactTypes = Literal["text", "table", "chart", "snowflake_query", "snowflake_python", "html"]

# WidgetParam.type
Literal["string", "text", "number", "integer", "boolean", "date", "ticker", "endpoint", "tabs"]

# FunctionCallSSEData.function
Literal[
    "get_widget_data", "get_extra_widget_data", "get_params_options",
    "add_widget_to_dashboard", "add_generative_widget", "update_widget_in_dashboard",
    "assign_tasks_to_agents", "execute_agent_tool", "manage_navigation_bar",
    "get_skill_content",
]
```
