# Build multi-step AI agent workflows with human approval at the right moments

An autonomous agent doing a 10-step task fully autonomously will eventually take a wrong path costing real money or trust; a state-machine agent with approval gates at the right steps gets the same speed with bounded risk.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/multi-step-workflows-with-approval) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agent runs a 10-step workflow end-to-end. Step 7 makes a wrong call. You don't notice until step 10 produces something embarrassing or expensive.

**After:** The workflow pauses at the steps where a wrong call would be costly, presents its decision to you for approval, and only proceeds after you confirm or correct.

## Tools used

cueapi, anthropic-api

## Pillar

[Foundations](https://thebuilderweekly.com/tutorials/pillars/foundations)
