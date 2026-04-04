# ADR-004: Three Equal Interfaces (CLI, UI, MCP)

## Status
Accepted

## Context
The system has three user-facing entry points:
- **CLI** (`bookie-breaker-cli`) — Terminal-based interface
- **UI** (`bookie-breaker-ui`) — Web dashboard
- **MCP Server** (`bookie-breaker-mcp-server`) — Model Context Protocol for LLM integration

We needed to decide whether one interface is primary (with others as secondary/limited) or all are first-class citizens.

## Decision
All three interfaces are equal, first-class entry points to the system. None is a subset of another. Each should be capable of accessing the full functionality of the system, adapted to its medium:

- **CLI:** Power-user workflows, scripting, automation, quick lookups
- **UI:** Visual dashboards, charts, data exploration, at-a-glance monitoring
- **MCP:** Natural language queries, LLM-assisted analysis, conversational interaction

All three interfaces communicate with the backend through the same agent/service APIs. No interface gets special access that others lack.

## Consequences

### Positive
- Users can pick the interaction mode that suits their context (terminal, browser, LLM chat)
- Forces clean API design — if three different consumers need the same data, the API must be well-designed
- MCP integration means AI assistants can leverage the full system

### Negative
- Three interfaces to build and maintain as a solo developer
- Must ensure feature parity or explicitly document which features are available where
- Testing surface area increases significantly

### Neutral
- Implementation can be staggered (CLI first as it's fastest to build, then UI, then MCP) while still designing for equality from the start
