# Verify every scheduled job actually ran

An agent can check a hundred cron outcomes per second against expected state, which a human doing daily ops review does in thirty minutes and misses silently.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/daily-cron-verification) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** You have ten cron jobs, each firing daily. When one fails silently you notice a week later when a customer emails.

**After:** An agent reads every cron run's outcome, compares it to expected state, and surfaces the failures within seconds of the scheduled fire time.

## Tools used

cueapi

## Pillar

[Accountability](https://thebuilderweekly.com/tutorials/pillars/accountability)
