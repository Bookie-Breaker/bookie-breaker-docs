# ADR-017: LLM Chat Interface Placement

## Status

Accepted

## Context

The UI includes an LLM analyst chat feature where users can ask questions about edges, games, and performance. We need to decide where it lives in the UI.

Options considered:

1. **Dedicated `/chat` page:** Full-page chat experience. Clear navigation, plenty of room for conversation history, but removes the user from the context they're asking about.
2. **Sidebar panel:** Collapsible panel available on every page. Keeps the user in context (viewing an edge while asking about it), but competes for screen space.
3. **Both:** Sidebar panel for quick context-aware questions, full page for extended conversations.

## Decision

Use a **persistent sidebar panel** available on every page, with no dedicated chat page.

The primary use case for the LLM analyst is context-aware questions: "Why does this edge exist?" while looking at the edges table, or "How has my CLV trended?" while viewing the performance dashboard. A sidebar panel keeps the user in the context they're asking about.

The sidebar is collapsible (closed by default) and opens via a button in the header or a keyboard shortcut. When opened, it takes ~350px of width on desktop. On tablet viewports, it overlays as a slide-out drawer.

The sidebar automatically includes page context in the LLM prompt: if the user is on `/edges` and asks "tell me more about the Lakers game," the agent receives the current edge data as context. If on `/predictions/123`, the specific prediction is included.

## Consequences

### Positive

- Context-aware by default — users don't need to copy/paste data into a chat to ask about it
- Always available without navigating away from the current view
- Simpler routing — no separate `/chat` page to maintain
- Natural interaction pattern: look at data, ask a question, see the answer alongside the data

### Negative

- Competes for horizontal screen space on smaller viewports
- Sidebar state (open/closed, conversation history) must be managed across page navigations
- Limited space for long conversations — extended back-and-forth may feel cramped

### Neutral

- If users request a full-page chat experience later, it can be added as a separate route without changing the sidebar
- Conversation history could optionally persist in localStorage or a backend store
