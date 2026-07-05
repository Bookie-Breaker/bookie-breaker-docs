# ADR-022: MCP Server SDK and Transport Selection

## Status

Accepted (2026-07-05, Phase 4)

## Context

Phase 4 builds the mcp-server: a stateless REST-to-MCP bridge exposing BookieBreaker capabilities as tools
for Claude Desktop, Claude Code, and other MCP clients. Two decisions were open:

1. **SDK:** the official `mcp` python-sdk bundles a frozen FastMCP 1.x snapshot; the standalone `fastmcp`
   package (2.x+) is the actively maintained successor by the same author.
2. **Transport:** the roadmap (written before mid-2025) called for "stdio and SSE" transports. The MCP
   specification deprecated the HTTP+SSE transport in its 2025-03-26 revision in favor of **Streamable
   HTTP**, and current clients (Claude Desktop/Code) speak streamable HTTP.

## Decision

- Use the **standalone `fastmcp` package** (which depends on the official `mcp` package for protocol
  compliance). It provides the in-memory `Client(server)` test harness (tools test without subprocesses),
  `http_app()` returning a plain ASGI app (uvicorn/Docker-friendly), `@custom_route` for the container
  healthcheck, and all transports behind one flag.
- Support **stdio** (default; Claude Desktop/Code launch `python -m mcp_server`) and **Streamable HTTP**
  (`MCP_TRANSPORT=http`, `/mcp` on port 8007, used in Docker Compose). **SSE is not wired**: it is
  deprecated by the spec; FastMCP retains `transport="sse"` as a one-line fallback should a legacy client
  ever require it.

The roadmap and component-spec wording "stdio and SSE" is superseded by "stdio and streamable HTTP".

## Consequences

### Positive

- In-memory client makes every tool CI-testable without Docker or subprocesses
- Streamable HTTP matches what current MCP clients actually negotiate
- One codebase serves both local (stdio) and containerized (HTTP) use

### Negative

- `fastmcp` moves faster than the official SDK — pin via Renovate and re-verify on major bumps
- Legacy SSE-only clients would need the documented one-line fallback enabled

### Neutral

- The server surface used (tool/resource decorators, `Client`, `http_app`) is stable API across recent
  FastMCP majors; a fallback to the official SDK's FastMCP would be mechanical if ever needed
