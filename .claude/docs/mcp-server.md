# OpenBB MCP Server

Source: https://pypi.org/project/openbb-mcp-server/ (fetched 2026-04-10)

## What It Is

Model Context Protocol (MCP) server that enables LLM agents to interact with OpenBB Platform's REST API endpoints. Provides dynamic tool discovery so agents activate only the tools they need.

## Installation

```bash
pip install openbb-mcp-server
```

Or with uvx:
```bash
uvx --from openbb-mcp-server --with openbb openbb-mcp
```

## Core Capabilities

- **Dynamic Tool Discovery** — agents browse categories and selectively activate tools (reduces token overhead)
- **Per-Session Tool Management** — multiple clients independently manage active toolsets
- **Tool Categories** — equity, crypto, economy, news, derivatives, etc. (based on installed extensions)

## Admin Tools

| Tool | Purpose |
|---|---|
| `available_categories` | Browse the tool category tree |
| `available_tools` | Inspect specific category tools |
| `activate_tools` | Enable tools for the session |
| `deactivate_tools` | Disable tools for the session |
| `list_prompts` | Access predefined workflow prompts |
| `execute_prompt` | Run a predefined prompt |

## Configuration

Supports customization via:
- Command-line arguments
- Environment variables
- JSON configuration files

Parameters: host/port binding, allowed categories, auth settings, system prompts.

## Integration with OpenBB Agents

To use MCP tools in a custom agent:

1. Set `"mcp-tools": True` in `agents.json` features
2. Access tools via `request.tools` in the query handler
3. Use `execute_agent_tool` function call to invoke tools
4. Tools execute on Workspace side, results come back as tool messages

See [agent-patterns.md](agent-patterns.md) Pattern 6 for implementation details.

## Architecture

```
LLM Agent  ←→  OpenBB MCP Server  ←→  OpenBB Platform REST API
                                            ↓
                                   Data Provider Extensions
                                   (Polygon, FMP, etc.)
```

The MCP server acts as middleware — translating MCP protocol calls into REST API requests against the OpenBB Platform.
