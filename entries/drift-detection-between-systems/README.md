# Catch drift between two systems of record before it becomes a bug

An agent reconciles two systems of record on every write with zero fatigue, catching silent divergence within a minute, where a human running a quarterly reconciliation script finds the same drift weeks after it happened.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/drift-detection-between-systems) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your billing system and your CRM drift out of sync. You find out during a quarterly audit and spend a week untangling which source was right.

**After:** An agent compares both systems on every significant write, surfaces mismatches within a minute, and logs the drift event with both sides' state captured.

## Tools used

cueapi, anthropic-api

## Pillar

[Accountability](https://thebuilderweekly.com/tutorials/pillars/accountability)
