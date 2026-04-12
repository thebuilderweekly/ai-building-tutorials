# Foundations

What every production agent needs underneath it. Observability, budget enforcement, retry logic, error handling, the stack you would otherwise reinvent for every project.

## Entries

_No entries yet. The first foundations tutorial is in the queue._

## The pattern

Every foundations tutorial teaches one piece of infrastructure that cuts across all agent types. The before-state is always "the agent works until it doesn't and you have no idea why." The after-state is always "the agent works, and when it breaks, you know exactly what happened."

- **Before-state:** "The agent runs in production and you're flying blind."
- **After-state:** "The agent runs in production and you see everything."

## Cluster tags in this pillar

- `logging` — structured event logging for agent pipelines
- `tracing` — distributed tracing across agent steps
- `metrics` — per-step latency and cost tracking
- `budgets` — spend-based throttling and circuit breakers
- `rate-limiting` — controlling agent API call velocity
- `cost-control` — keeping agent operations within budget
- `orchestration` — multi-step workflow management
- `human-in-the-loop` — approval gates in automated pipelines
- `state-machines` — explicit state transitions for complex flows
