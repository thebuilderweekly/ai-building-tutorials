# Send context-aware Slack notifications that get noticed instead of muted

An agent that calibrates notification urgency per event reaches the right person at the right time with eighty percent lower mute rate than a fixed-rule notification system.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/context-aware-slack-notifications) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your monitoring system posts every event to a Slack channel. The channel fires fifty times a day. Your team muted it three weeks ago. Now critical alerts go unread.

**After:** An agent classifies each event by severity, audience, and timeliness. Critical events DM the on-call engineer. Medium events post to a channel. Low events batch into an hourly digest. Mute rates collapse.

## Tools used

slack-api, anthropic-api

## Pillar

[Communication](https://thebuilderweekly.com/tutorials/pillars/communication)
