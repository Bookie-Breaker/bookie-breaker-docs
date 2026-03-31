# PLANNING: cli

## Service
- **Name:** cli
- **Language:** Go 1.22+
- **Framework:** Cobra + Charm ecosystem (Bubble Tea, Lip Gloss, Glamour)

## Implementation Phase
Phase 3 (First Interface & Paper Trading)

## Purpose
Terminal-based interface for interacting with the BookieBreaker system. Provides commands for viewing edges, placing paper bets, checking performance, querying lines and stats, and asking the LLM analyst questions. Designed for quick lookups and automation/scripting.

## Ordered Task List

1. Initialize Go module, set up project structure: `cmd/cli/`, `internal/`, `pkg/`
2. Set up Cobra root command with global flags: `--format` (table/json), `--sport`, `--config`
3. Implement configuration management: read config from `~/.bookiebreaker/config.yaml` (service URLs, default sport, preferences)
4. Implement HTTP client layer: shared client for calling agent, lines-service, statistics-service, bookie-emulator
5. Implement `bb edges` command:
   - Call agent `GET /api/v1/edges`
   - Flags: `--sport`, `--bet-type`, `--min-edge`
   - Display as formatted table with Lip Gloss styling
6. Implement `bb slate` command:
   - Call agent `GET /api/v1/slate`
   - Show today's games with predictions across leagues
7. Implement `bb predict <game>` command:
   - Call agent for prediction details
   - Show calibrated probabilities, confidence intervals, feature importance
8. Implement `bb lines <game>` command:
   - Call lines-service directly `GET /api/v1/lines/current`
   - Show odds across sportsbooks in formatted table
9. Implement `bb bet place` command:
   - Interactive or flag-based bet placement
   - Call bookie-emulator `POST /api/v1/bets`
10. Implement `bb bet list` command:
    - Call bookie-emulator `GET /api/v1/bets`
    - Flags: `--sport`, `--status`, `--bet-type`, `--limit`
11. Implement `bb performance` command:
    - Call bookie-emulator `GET /api/v1/performance`
    - Show ROI, win rate, CLV, units in styled output
12. Implement `bb ask <question>` command (Phase 4 dependency):
    - Call agent `POST /api/v1/analyze`
    - Stream or display LLM response with Glamour markdown rendering
13. Implement `bb pipeline run` command:
    - Call agent `POST /api/v1/pipeline/run`
    - Show pipeline status
14. Style all output with Lip Gloss (colors, borders, padding) and Glamour (markdown rendering)
15. Add shell completion generation (bash, zsh, fish)
16. Write unit tests for output formatting and flag parsing
17. Create Dockerfile (for Docker-based usage) and standalone binary build
18. Add `.env.example`

## Dependencies
- **agent** (Phase 3) for edges, slate, pipeline, and analysis endpoints
- **lines-service** (Phase 1) for direct line lookups
- **statistics-service** (Phase 1) for direct stat lookups
- **bookie-emulator** (Phase 3) for bet placement and performance
- **agent LLM integration** (Phase 4) for the `ask` command

## Complexity
**M** -- Primarily an API client with formatted output. Cobra and the Charm ecosystem handle the hard parts. The main work is in building clean, readable terminal output for varied data types.

## Definition of Done
- [ ] `bb edges` displays NBA edges in a polished terminal table
- [ ] `bb slate` shows today's games with predictions
- [ ] `bb bet place` successfully places a paper bet
- [ ] `bb bet list` shows bet history with filters
- [ ] `bb performance` displays ROI, win rate, CLV metrics
- [ ] `bb lines <game>` shows odds across sportsbooks
- [ ] `bb ask` sends a question and displays the LLM response (Phase 4)
- [ ] All commands support `--format json` for scripting
- [ ] Shell completion works for bash/zsh
- [ ] Binary builds for Linux and macOS

## Key Documentation
- [CLI Component](../bookie-breaker-docs/components/cli.md)
- [Feature Inventory: CLI-001 through CLI-016](../bookie-breaker-docs/architecture/feature-inventory.md)
- [API Contracts](../bookie-breaker-docs/api-contracts/)
