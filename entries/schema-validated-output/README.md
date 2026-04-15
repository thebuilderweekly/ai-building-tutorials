# Force structured agent output with JSON schema validation and retry

An agent with strict schema validation retries until it produces valid output, which a human reviewing free-form output would catch by reading every response and re-prompting manually.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/schema-validated-output) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agent returns prose when you need structured data. You add a JSON parser. The parser fails twenty percent of the time on missing fields, extra commas, or hallucinated keys.

**After:** Every agent response is validated against a JSON schema. Invalid responses trigger an automatic retry with the schema and the validation error appended to the prompt. The agent converges to valid output every time.

## Tools used

anthropic-api

## Pillar

[Scoping](https://thebuilderweekly.com/tutorials/pillars/scoping)
