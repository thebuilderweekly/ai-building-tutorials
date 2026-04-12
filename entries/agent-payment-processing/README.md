# Process payments from your AI agent without losing money to retries

An agent retrying a charge without idempotency keys has charged the same customer 47 times before a human noticed; with proper idempotency the same agent can safely retry indefinitely.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/agent-payment-processing) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agent triggers a charge, gets a network timeout, retries, and now your customer is charged twice. You discover this through a chargeback notice three days later.

**After:** The agent retries any failed charge as many times as it wants, with idempotency keys ensuring each logical payment results in exactly one actual charge regardless of network conditions.

## Tools used

stripe, anthropic-api

## Pillar

[Payments](https://thebuilderweekly.com/tutorials/pillars/payments)
