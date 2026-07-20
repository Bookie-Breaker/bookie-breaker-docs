# Operator Playbooks

Task-oriented guides for running BookieBreaker day-to-day: installing the workspace, operating the stack,
finding and betting edges, managing seasons, and recovering when something breaks. Each playbook is
self-contained — you can follow it start-to-finish without leaving the page — and links into
[operations/](../operations/) for design background.

The [operations/](../operations/) docs explain how the system is built and why; these playbooks explain
what to type when you want to do something.

## Which playbook do I need?

| Situation                                           | Playbook                                                          |
| --------------------------------------------------- | ----------------------------------------------------------------- |
| Fresh machine, nothing installed                    | [01 — Installation](01-installation.md)                           |
| Start/stop the stack, check health, read logs       | [02 — Daily Operations](02-daily-operations.md)                   |
| Review the slate, inspect edges, place paper bets   | [03 — Finding and Betting Edges](03-finding-and-betting-edges.md) |
| Player props, parlays, or live in-game betting      | [04 — Parlays, Props, and Live](04-parlays-props-and-live.md)     |
| Run the prediction pipeline or manage its schedules | [05 — Pipeline and Scheduling](05-pipeline-and-scheduling.md)     |
| A league's season is starting or ending             | [06 — Seasonal Operations](06-seasonal-operations.md)             |
| Something is broken                                 | [07 — Troubleshooting](07-troubleshooting.md)                     |
| Dashboards, metrics, traces, and logs in depth      | [08 — Observability](08-observability.md)                         |
| Updates, rebuilds, model retraining, periodic care  | [09 — Maintenance](09-maintenance.md)                             |

## Conventions

- Commands run from the `BookieBreaker/` workspace root unless a playbook says otherwise.
- `bb` is the CLI binary built from
  [bookie-breaker-cli](https://github.com/Bookie-Breaker/bookie-breaker-cli); playbook 01 covers building it
  and putting it on your PATH.
- `task` is [go-task](https://taskfile.dev), resolved through the root `Taskfile.yml` symlink.
- Service URLs assume the default local ports (see the port table in
  [02 — Daily Operations](02-daily-operations.md#2-service-and-port-reference)).
