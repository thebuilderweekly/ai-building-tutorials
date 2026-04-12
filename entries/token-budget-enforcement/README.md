# Stop your AI agent from running up a thousand-dollar bill overnight

An agent looping on a malformed input can burn through a month's API budget in 20 minutes; a budget-enforced agent stops itself before the bill arrives.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/token-budget-enforcement) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agent hits an edge case that puts it in a retry loop. By morning you have a thousand-dollar Anthropic invoice and an angry email from your CFO.

**After:** The agent tracks cumulative spend per session and per day, throttles when approaching limits, switches to cheaper models when over thresholds, and stops entirely before damaging your bill.

## Tools used

anthropic-api

## Pillar

[Foundations](https://thebuilderweekly.com/tutorials/pillars/foundations)
