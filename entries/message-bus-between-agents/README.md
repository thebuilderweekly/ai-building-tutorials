# Wire agents together with a message bus instead of function calls

A message-bus-coordinated agent system handles stage failures, retries, and back-pressure correctly on every single run, where a human wiring the same coordination by hand would cut corners on the retry logic and regret it in production.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/message-bus-between-agents) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agents talk to each other through nested function calls. One stage's failure cascades through the stack and you lose state.

**After:** Every agent publishes its output to a bus, every consumer subscribes to what it needs, and failures in one agent don't corrupt state in another.

## Tools used

inngest, anthropic-api

## Pillar

[Agent Teams](https://thebuilderweekly.com/tutorials/pillars/agent-teams)
