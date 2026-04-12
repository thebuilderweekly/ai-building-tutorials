# Generate images from your AI agent without burning a hundred dollars on bad outputs

A naive image generation agent produces 100 mediocre images at $5 total cost; a properly-prompted agent with output verification produces 10 great images at the same cost.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/agent-image-generation) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agent generates images for every blog post but most of them are bad. You're paying for hundreds of generations to get a few usable results.

**After:** The agent uses a structured prompt template, generates with controlled variations, runs each output through a quick quality check, and only keeps results that pass.

## Tools used

replicate, anthropic-api

## Pillar

[Operational](https://thebuilderweekly.com/tutorials/pillars/operational)
