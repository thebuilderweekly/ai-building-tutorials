# Run a roster of four agents in a single workflow without the wheels coming off

An agent team with well-defined handoffs produces the same multi-step result end-to-end in one minute that a four-person human team takes a full meeting to coordinate, and the agent team doesn't need a meeting.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/roster-of-four-agents) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** You tried to chain four agents together and the output of each one contaminated the input of the next until the final agent produced garbage.

**After:** Four agents run in a defined sequence with typed handoffs, each stage produces verifiable intermediate output, and you can inspect any stage independently if the final result looks wrong.

## Tools used

anthropic-api, inngest

## Pillar

[Agent Teams](https://thebuilderweekly.com/tutorials/pillars/agent-teams)
