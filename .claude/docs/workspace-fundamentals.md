# OpenBB Workspace Fundamentals

Source: https://docs.openbb.co/workspace (fetched 2026-04-10)

## What is Workspace?

Secure application for enterprise AI workflows. Integrates data management, customizable interfaces, and AI. Supports cloud and on-premises deployment.

## Core Architecture

### Widgets — the building blocks

Every widget has four layers:
1. **Data connectivity** — internal or external source
2. **Metadata** — title, description, category, sub-category, source
3. **Visual presentation** — tables (AgGrid), charts (Plotly), PDFs, images, feeds
4. **Parameters** — configurable inputs for user interaction

### Parameter Binding

Grouped parameters sync across widgets. When a user changes a ticker in one widget, all widgets sharing that parameter group update automatically.

**Group naming**: Must use `"Group 1"`, `"Group 2"`, etc. Custom names fail silently.

### Dashboards

Blank-canvas workspaces combining:
- Dynamic data widgets (live)
- Static resources (PDFs, images, documents, spreadsheets)
- AI-generated artifacts (from agent chat)
- Annotations and notes

Can be shared across organizations.

### Apps

Pre-built dashboard templates with:
- Curated widget sets with linked parameters
- Pre-selected AI agents with custom prompts
- Synchronized updates across all connected widgets

Example categories: portfolio management, market surveillance, research.

### AI Agents

Agents leverage widget metadata for:
- Contextual queries across dashboard data
- Multi-step analysis and calculations
- Dual-mode operation: reactive (query-specific) + proactive (monitoring)
- Artifact generation (tables, charts, text) that feed back into dashboards
- Citation support linking answers to source widgets

## Widget Priority Levels (for agents)

Agents receive three widget groups via `QueryRequest.widgets`:
- **Primary** — explicitly selected by user (highest priority)
- **Secondary** — present on the active dashboard
- **Extra** — all widgets available in the workspace (not yet fully supported)

## Key Principles

1. **Data-First Design** — widgets are the fundamental units of composition
2. **Stateless Agents** — every request includes full conversation history and context
3. **SSE Streaming** — all agent responses stream via Server-Sent Events
4. **Context Preservation** — agents maintain awareness via the full QueryRequest
5. **Artifact Feedback Loops** — agent outputs integrate back into dashboards
