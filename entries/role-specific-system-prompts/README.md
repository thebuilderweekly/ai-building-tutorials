# Author role-specific system prompts that actually change behavior

An agent with a well-scoped role-specific system prompt produces deterministically specialized output on every call, where a human generalist switching between roles loses context and degrades over the day.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/role-specific-system-prompts) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** You have one generic agent handling researcher, writer, and editor roles, and the output reads like all three are fighting for the keyboard.

**After:** Three role-specific agents each produce their specialized output, the composition is clean, and changing any one prompt doesn't corrupt the others.

## Tools used

anthropic-api

## Pillar

[Agent Teams](https://thebuilderweekly.com/tutorials/pillars/agent-teams)
