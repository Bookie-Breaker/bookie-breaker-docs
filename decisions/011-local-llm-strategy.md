# ADR-011: Local LLM Strategy

## Status
Accepted

## Context
The agent and MCP server rely on LLM capabilities for edge analysis, game previews, daily summaries, and natural language Q&A. The initial plan assumed Anthropic API as the sole LLM provider. However:
- Cloud API costs accumulate during development and testing (prompt iteration, integration testing, daily summaries)
- Offline development and experimentation require no external dependencies
- Self-hosted models enable cost-free bulk operations (e.g., analyzing entire slates, generating training data annotations)
- Future fine-tuning workflows benefit from a local inference environment
- Data privacy: keeping prediction/betting data local during analysis is preferable

## Decision

### Provider Abstraction Layer

The agent implements a configurable LLM provider interface that supports multiple backends. Switching providers is config-only — no code changes required.

**Configuration:**
- `LLM_PROVIDER` — `anthropic` (default) or `ollama`
- `LLM_BASE_URL` — provider endpoint (default: `https://api.anthropic.com` for Anthropic, `http://ollama:11434` for Ollama)
- `LLM_MODEL` — model identifier (e.g., `claude-sonnet-4-5-20250514` for Anthropic, `llama3.1:8b` for Ollama)

### Local LLM: Ollama

Ollama is chosen for local LLM hosting because:
- Simple Docker deployment (single container, no complex setup)
- Broad model support (Llama, Mistral, Gemma, Phi, etc.)
- OpenAI-compatible API (`/api/chat`) — easy to integrate alongside Anthropic SDK
- GPU passthrough support via Docker Compose overlay
- Model management CLI (`ollama pull`, `ollama list`)

### Infrastructure

- **Docker Compose:** Ollama container included in base `docker-compose.yml` with configurable model pull on first start
- **GPU support:** Optional `docker-compose.gpu.yml` overlay for NVIDIA GPU passthrough
- **Health check:** Ollama exposes a health endpoint; container marked healthy when ready to serve
- **Model selection:** Default to a small model (e.g., `phi3:mini` ~2GB) for development; document recommended models for production-quality analysis

### Tiered LLM Usage

| Use Case | Recommended Provider | Rationale |
|---|---|---|
| Development & prompt iteration | Ollama (local) | Zero cost, fast iteration, no rate limits |
| Routine summaries & alerts | Ollama (local) or Anthropic Haiku | Cost-efficient for high-volume, low-complexity tasks |
| Detailed edge analysis | Anthropic Sonnet/Opus | Highest quality for nuanced sports analysis |
| Bulk annotation for training data | Ollama (local) | Cost-free processing of historical data |

## Consequences

### Positive
- Zero-cost LLM during development and testing
- No external dependency for basic LLM features
- Foundation for future fine-tuning workflows
- Provider flexibility — can add new backends (OpenAI, etc.) via the same abstraction

### Negative
- Local model quality is lower than Anthropic's models — not suitable for all use cases
- GPU hardware needed for reasonable inference speed on larger models
- Additional Docker Compose complexity (Ollama container, model volume, GPU overlay)
- Need to maintain prompt compatibility across different model formats

### Neutral
- Ollama's API is similar but not identical to Anthropic's — the provider abstraction handles translation
- Model files are large (2-8 GB) and stored in a Docker volume
