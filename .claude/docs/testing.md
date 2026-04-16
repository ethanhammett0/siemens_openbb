# Testing OpenBB Agents

Source: openbb-ai/openbb_ai/testing.py, agents-for-openbb/testing/ (read 2026-04-10)

## SDK Testing Module

The `openbb_ai.testing` module provides `CopilotResponse` for parsing and asserting on SSE event streams.

### CopilotResponse

```python
from openbb_ai.testing import CopilotResponse

response = CopilotResponse(event_stream_text)

# Access parsed content
response.text                 # Combined text from all copilotMessageChunk events
response.function_calls       # List of copilotFunctionCall events
response.citations            # List of copilotCitationCollection events
response.events               # All parsed events
```

### Fluent Assertion API

```python
response = CopilotResponse(event_stream)

# Chain assertions about event order and content:
response.starts("copilotStatusUpdate") \
    .with_("Starting analysis") \
    .then("copilotMessage") \
    .and_("some expected text") \
    .then("copilotCitationCollection") \
    .ends("copilotCitationCollection")

# Check any event contains specific content:
response.has_any("copilotMessage", "expected substring")
response.has_any("copilotStatusUpdate", {"eventType": "INFO"})
```

### Parsed Event Types

| SSE Event | Parsed As |
|---|---|
| `copilotMessageChunk` | Accumulated into single `copilotMessage` event |
| `copilotMessageArtifact` | Individual event with JSON content |
| `copilotFunctionCall` | Individual event with function call data |
| `copilotStatusUpdate` | Individual event with status data |
| `copilotCitationCollection` | Individual event with citation data |

## Test Payloads

The `agents-for-openbb/testing/test_payloads/` directory contains JSON fixtures for testing:

### single_message.json — Basic human message
```json
{
  "messages": [{"role": "human", "content": "what is 1 + 1?"}]
}
```

### message_with_primary_widget.json — Message + widget context
```json
{
  "messages": [{"role": "human", "content": "Fetch the latest news..."}],
  "widgets": {
    "primary": [{
      "uuid": "123e4567-e89b-12d3-a456-426614174000",
      "origin": "openbb",
      "widget_id": "company_news",
      "name": "Company News",
      "description": "News about a company",
      "params": [{"name": "ticker", "type": "string", "current_value": "AAPL"}],
      "metadata": {}
    }]
  }
}
```

### Other available payloads:
- `message_with_context.json` — message with context items
- `message_with_primary_widget_and_tool_call.json` — widget data retrieval flow
- `message_with_primary_widget_and_tool_call_pdf_url.json` — PDF widget flow
- `multiple_messages.json` — multi-turn conversation
- `retrieve_widget_from_dashboard.json` — dashboard widget access
- `retrieve_widget_from_dashboard_with_result.json` — with tool results

## Writing Agent Tests

Pattern from the example agents:

```python
import httpx
import pytest
from openbb_ai.testing import CopilotResponse

@pytest.mark.asyncio
async def test_basic_query():
    async with httpx.AsyncClient(base_url="http://localhost:8000") as client:
        payload = {"messages": [{"role": "human", "content": "Hello"}]}
        response = await client.post("/v1/query", json=payload)
        assert response.status_code == 200

        copilot_response = CopilotResponse(response.text)
        assert len(copilot_response.text) > 0

@pytest.mark.asyncio
async def test_widget_data_retrieval():
    # Load test payload
    with open("testing/test_payloads/message_with_primary_widget.json") as f:
        payload = json.load(f)

    async with httpx.AsyncClient(base_url="http://localhost:8000") as client:
        response = await client.post("/v1/query", json=payload)
        copilot_response = CopilotResponse(response.text)

        # Should return a function call to get widget data
        assert len(copilot_response.function_calls) > 0
        copilot_response.starts("copilotFunctionCall") \
            .with_({"function": "get_widget_data"})
```
