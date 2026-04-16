# Agent Implementation Patterns

Source: https://github.com/OpenBB-finance/agents-for-openbb — all examples read 2026-04-10

These patterns are distilled from 12+ official OpenBB agent examples. Each section shows the core pattern with working code from the repo.

---

## Pattern 1: Basic Streaming Agent (Example 31)

Simplest agent — receives messages, streams LLM response.

```python
@app.post("/v1/query")
async def query(request: QueryRequest) -> EventSourceResponse:
    openai_messages = [
        ChatCompletionSystemMessageParam(role="system", content="You are a financial assistant.")
    ]
    for message in request.messages:
        if message.role == "human":
            openai_messages.append(ChatCompletionUserMessageParam(role="user", content=message.content))
        elif message.role == "ai" and isinstance(message.content, str):
            openai_messages.append(ChatCompletionAssistantMessageParam(role="assistant", content=message.content))

    async def execution_loop():
        # Reasoning steps before response
        yield reasoning_step("Starting analysis...", event_type="INFO").model_dump()
        yield reasoning_step("Step with details", event_type="INFO",
                             details={"key1": "value1"}).model_dump()
        # Stream LLM
        client = openai.AsyncOpenAI()
        async for event in await client.chat.completions.create(
            model="gpt-4o", messages=openai_messages, stream=True
        ):
            if chunk := event.choices[0].delta.content:
                yield message_chunk(chunk).model_dump()

    return EventSourceResponse(content=execution_loop(), media_type="text/event-stream")
```

---

## Pattern 2: Widget Data Retrieval + Citations (Examples 30, 32)

Two-phase: first request fetches data, second request processes it.

```python
@app.post("/v1/query")
async def query(request: QueryRequest) -> EventSourceResponse:
    last_message = request.messages[-1]

    # PHASE 1: If human message + primary widgets, request data
    orchestration_requested = (
        last_message.role == "ai" and last_message.agent_id == "openbb-copilot"
    )
    if (
        (last_message.role == "human" or orchestration_requested)
        and request.widgets and request.widgets.primary
    ):
        widget_requests = []
        for widget in request.widgets.primary:
            widget_requests.append(WidgetRequest(
                widget=widget,
                input_arguments={p.name: p.current_value for p in widget.params},
            ))
        async def retrieve():
            yield get_widget_data(widget_requests).model_dump()
        return EventSourceResponse(content=retrieve(), media_type="text/event-stream")

    # PHASE 2: Process tool response (widget data came back)
    context_str = ""
    citations_list = []
    for index, message in enumerate(request.messages):
        if message.role == "human":
            openai_messages.append(...)
        elif message.role == "tool" and index == len(request.messages) - 1:
            # Extract widget data
            for result in message.data:
                for item in result.items:
                    context_str += f"{item.content}\n"

            # Build citations
            for widget_data_request in message.input_arguments["data_sources"]:
                filtered = [w for w in request.widgets.primary
                           if str(w.uuid) == widget_data_request["widget_uuid"]]
                if filtered:
                    citations_list.append(cite(
                        widget=filtered[0],
                        input_arguments=widget_data_request["input_args"],
                    ))

    # Append context to last user message, stream response
    async def execution_loop():
        # ... stream LLM response ...
        if citations_list:
            yield citations(citations_list).model_dump()

    return EventSourceResponse(content=execution_loop(), media_type="text/event-stream")
```

**Key insight**: Only process `message.role == "tool"` at `index == len(request.messages) - 1` to avoid context buildup from old widget data.

---

## Pattern 3: Inline Charts & Tables (Examples 33, 34)

Yield artifacts after the LLM response:

```python
async def execution_loop():
    # Stream LLM text first...
    async for event in await client.chat.completions.create(...):
        yield message_chunk(chunk).model_dump()

    # Then yield artifacts
    yield chart(
        type="line",  # line | bar | scatter | pie | donut
        data=[{"x": 1, "y": 2}, {"x": 2, "y": 3}],
        x_key="x", y_keys=["y"],
        name="Price Chart", description="Stock price over time",
    ).model_dump()

    yield table(
        data=[{"ticker": "AAPL", "price": 150.0}, {"ticker": "GOOGL", "price": 140.0}],
        name="Holdings", description="Current positions",
    ).model_dump()
```

**Chart types:**
- `line`, `bar`, `scatter` → use `x_key` + `y_keys`
- `pie`, `donut` → use `angle_key` + `callout_label_key`

---

## Pattern 4: PDF Handling with Citations (Example 35, 36)

PDFs come as either base64 content or URL references:

```python
async def handle_widget_data(data):
    for result in data:
        for item in result.items:
            if isinstance(item.data_format, PdfDataFormat):
                if isinstance(item, SingleDataContent):
                    file_content = base64.b64decode(item.content)
                elif isinstance(item, SingleFileReference):
                    async with httpx.AsyncClient() as client:
                        file_content = (await client.get(str(item.url))).content
                with pdfplumber.open(io.BytesIO(file_content)) as pdf:
                    for page in pdf.pages:
                        text += page.extract_text() + "\n\n"
```

PDF citations support bounding boxes for highlighting:
```python
citation = cite(widget=widget, input_arguments=input_args)
citation.quote_bounding_boxes = [[
    CitationHighlightBoundingBox(text="highlight text", page=1, x0=72, top=117, x1=259, bottom=135)
]]
```

---

## Pattern 5: Custom Feature Toggles (Example 37)

Define custom toggles in `agents.json`, check via `workspace_options`:

```python
# In agents.json features:
"features": {
    "streaming": True,
    "deep-research": {"label": "Deep Research", "default": False, "description": "..."},
    "web-search": {"label": "Web Search", "default": True, "description": "..."},
}

# In query handler:
workspace_options = getattr(request, "workspace_options", [])
deep_research_enabled = "deep-research" in workspace_options
web_search_enabled = "web-search" in workspace_options
```

---

## Pattern 6: MCP Tool Integration (Example 38)

Enable with `"mcp-tools": True` in features. Tools come via `request.tools`.

```python
# Build OpenAI function from MCP tool definition
functions = [{
    "name": "execute_agent_tool",
    "parameters": {
        "type": "object",
        "properties": {
            "server_id": {"type": "string", "enum": [t.server_id for t in request.tools]},
            "tool_name": {"type": "string", "enum": [t.name for t in request.tools]},
            "parameters": {"type": "object", "additionalProperties": True},
        },
        "required": ["server_id", "tool_name", "parameters"],
    }
}]

# When LLM calls the function, yield it back to Workspace for execution:
yield FunctionCallSSE(data=FunctionCallSSEData(
    function="execute_agent_tool",
    input_arguments={"server_id": sid, "tool_name": name, "parameters": params},
    extra_state={"copilot_function_call_arguments": {...}},
)).model_dump()
```

---

## Pattern 7: Dashboard Widget Access (Example 40)

Enable `widget-dashboard-search: True` to access secondary widgets:

```python
all_widgets = []
if request.widgets:
    if request.widgets.primary:
        all_widgets.extend(request.widgets.primary)    # Explicitly selected
    if request.widgets.secondary:
        all_widgets.extend(request.widgets.secondary)  # Dashboard widgets

# Access tab structure via workspace_state:
if request.workspace_state and request.workspace_state.current_dashboard_info:
    tabs = request.workspace_state.current_dashboard_info.tabs
    active_tab = request.workspace_state.current_dashboard_info.current_tab_id
```

---

## Pattern 8: Dynamic Skill Loading (Example 41)

Agents can request skill content from a catalog:

```python
class SkillQueryRequest(QueryRequest):
    skills_catalog: list[SkillCatalogEntry] | None = None
    selected_skills: list[SkillPayload] | None = None

# If skill active, inject into system prompt:
if active_skill:
    system_content += f"""
<user-authored-skill-content name="{active_skill.slug}">
{active_skill.content_markdown}
</user-authored-skill-content>"""

# To request a skill, yield a function call:
yield FunctionCallSSE(data=FunctionCallSSEData(
    function="get_skill_content",
    input_arguments={"slug": slug},
)).model_dump()
```

---

## Pattern 9: HTML Artifacts (Example 39)

Return custom HTML rendered inline. SDK doesn't natively support `type="html"` yet — construct manually:

```python
def html_artifact(content: str, name: str, description: str) -> dict:
    return {
        "event": "copilotMessageArtifact",
        "data": json.dumps({
            "type": "html",
            "uuid": str(uuid.uuid4()),
            "name": name,
            "description": description,
            "content": content,
        }),
    }
```

Users can convert HTML artifacts into dashboard widgets.

---

## Pattern 10: Non-OpenAI LLM via OpenRouter (Example 99)

```python
url = "https://openrouter.ai/api/v1/chat/completions"
headers = {"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"}
data = {"model": "deepseek/deepseek-chat-v3-0324", "messages": messages, "stream": True}

async with httpx.AsyncClient(timeout=60.0) as client:
    async with client.stream("POST", url, headers=headers, json=data) as response:
        async for line in response.aiter_lines():
            if line.startswith("data: "):
                chunk = json.loads(line[6:])
                content = chunk["choices"][0]["delta"].get("content")
                if content:
                    yield message_chunk(content).model_dump()
```

---

## CORS Origins

Always include:
```python
allow_origins=["https://pro.openbb.co"]
```
For development, add `"http://localhost:1420"` or `"*"`.
