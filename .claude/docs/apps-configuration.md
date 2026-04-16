# Apps Configuration (apps.json)

Source: https://docs.openbb.co/workspace/developers/apps (fetched 2026-04-10)

## Purpose

Apps combine widgets, prompts, and AI agents into pre-built dashboard templates. Configured via `apps.json` served alongside `widgets.json`.

## apps.json Structure

```json
[
  {
    "name": "My Research App",
    "description": "Equity research terminal for fintech coverage",
    "img": "https://example.com/app-icon.png",
    "img_dark": "https://example.com/app-icon-dark.png",
    "img_light": "https://example.com/app-icon-light.png",
    "allowCustomization": true,
    "tabs": [...],
    "groups": [...],
    "prompts": [...]
  }
]
```

## Tabs

Each tab is a dashboard view with positioned widgets:

```json
{
  "tabs": [
    {
      "id": "overview",
      "name": "Overview",
      "layout": [
        {
          "i": "widget_endpoint_id",
          "x": 0,
          "y": 0,
          "w": 20,
          "h": 9,
          "state": {
            "params": {
              "symbol": "AAPL",
              "start_date": "2024-01-01"
            },
            "chartView": {
              "enabled": true,
              "chartType": "line"
            }
          }
        }
      ]
    }
  ]
}
```

### Layout Grid

- `i` — widget ID (must match key in widgets.json)
- `x`, `y` — position on grid (left/top)
- `w`, `h` — width and height
- `state.params` — initial parameter values
- `state.chartView` — chart display settings

## Groups (Parameter Binding)

Groups synchronize parameters across multiple widgets:

```json
{
  "groups": [
    {
      "name": "Group 1",
      "type": "param",
      "paramName": "symbol",
      "defaultValue": "AAPL",
      "widgetIds": ["widget_a", "widget_b", "widget_c"]
    },
    {
      "name": "Group 2",
      "type": "endpointParam",
      "paramName": "start_date",
      "defaultValue": "2024-01-01",
      "widgetIds": ["widget_a", "widget_b"]
    }
  ]
}
```

### Group Properties

| Field | Description |
|---|---|
| `name` | **Must be "Group 1", "Group 2", etc.** Custom names fail silently |
| `type` | `"param"` (regular) or `"endpointParam"` (endpoint-derived) |
| `paramName` | The parameter to synchronize |
| `defaultValue` | Initial value |
| `widgetIds` | Array of widget IDs that share this parameter |

**Critical**: When user changes the grouped parameter in ANY widget, ALL widgets in the group update simultaneously.

## Prompts

Pre-defined AI query suggestions shown in the app:

```json
{
  "prompts": [
    "Summarize the portfolio performance for the current period",
    "What are the top contributors to portfolio returns?",
    "Analyze sector allocation and recommend adjustments"
  ]
}
```

## Relationship: widgets.json ↔ apps.json

- **widgets.json** defines WHAT each widget is (data source, params, display)
- **apps.json** defines WHERE widgets go and HOW they're connected (tabs, layout, groups)
- Every widget referenced in apps.json layout MUST have a matching entry in widgets.json
- A widget can exist in widgets.json without being in apps.json (available but not pre-placed)
