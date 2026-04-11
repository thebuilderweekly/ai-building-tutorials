# Agent Teams

Running coordinated multi-agent systems. Tutorials in this pillar teach patterns for composing multiple agents into a workflow where each one does a specialized job and the handoffs don't corrupt state.

## Entries

_No entries yet. The first agent-teams tutorial is in the queue — see [queue/topics.json](../../queue/topics.json)._

## The pattern

Every agent-teams tutorial follows the same arc: a task is too big for one agent, the task is split into specialized roles, each role runs against its own scoped prompt, the outputs flow through typed handoffs, and the whole thing can be inspected stage-by-stage if the final result looks wrong.

- **Before-state:** "One generic agent is trying to play three roles and the output is incoherent."
- **After-state:** "Three role-specific agents run in sequence with clean handoffs, each stage is inspectable, and changing one prompt doesn't break the others."

The load-bearing question in every agent-teams tutorial is "what's the handoff contract between stages, and how do you catch a bad handoff early?" A tutorial that chains four agents without naming the typed contract between them isn't finished — it's just a chain of function calls waiting to fail silently.

## Cluster tags in this pillar

- `multi-agent` — entries that run more than one agent in a workflow
- `orchestration` — entries centered on the orchestrator layer
- `workflow` — entries that define an end-to-end workflow shape
- `system-prompts` — entries focused on role-specific prompt design
- `roles` — entries about agent specialization
- `agent-specialization` — entries about scoping each agent's job
- `message-bus` — entries that use a bus instead of direct calls
- `coordination` — entries about cross-agent coordination
- `async` — entries about asynchronous agent coordination
