# Build a research agent team where one agent finds, one verifies, one summarizes

Three specialized agents working in sequence produce verified research in five minutes; one generalist agent doing all three jobs takes twenty minutes and produces ungrounded claims.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/research-agent-team) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** You ask one agent to research a topic and it returns a confident summary mixing real facts, half-remembered training data, and outright fabrications, with no way to tell which is which.

**After:** Three specialized agents run in sequence. The Researcher fetches primary sources. The Verifier checks every claim against those sources. The Summarizer writes only what survived verification.

## Tools used

anthropic-api

## Pillar

[Agent Teams](https://thebuilderweekly.com/tutorials/pillars/agent-teams)
