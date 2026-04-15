# Route customer queries to specialized agents based on intent classification

A router that dispatches to five specialists handles a thousand customer queries per hour with consistent expertise per category, which a single generalist agent handles slower and worse.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/intent-routing-agent-team) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your support agent answers every kind of question with the same generalist tone. Billing questions get vague refund policies. Technical questions get product marketing. Customers escalate to humans for clarity.

**After:** A router classifies every incoming query and dispatches it to a specialist agent with the right tools, knowledge scope, and tone. Billing questions get billing-trained responses. Technical questions get debugging-trained responses.

## Tools used

anthropic-api

## Pillar

[Agent Teams](https://thebuilderweekly.com/tutorials/pillars/agent-teams)
