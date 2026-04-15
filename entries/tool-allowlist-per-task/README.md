# Constrain agent tool access with per-task allowlists

An agent with fifty tools available picks the wrong one twelve percent of the time; an agent with five tools scoped to its current task picks correctly ninety-nine percent of the time.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/tool-allowlist-per-task) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agent has access to every tool in your codebase. When asked to send an email, it sometimes calls the database directly. When asked to read data, it sometimes attempts to write. The error rate climbs as the tool surface grows.

**After:** Each task type declares which tools it is allowed to use. The agent loads only those tools at runtime. Tool selection errors drop to near zero because the wrong tool is not in the menu.

## Tools used

anthropic-api

## Pillar

[Scoping](https://thebuilderweekly.com/tutorials/pillars/scoping)
