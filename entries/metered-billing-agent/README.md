# Meter agent usage and bill customers in real-time without manual reconciliation

An agent meters a thousand customer API calls per second with sub-cent precision, which a manual usage report at month-end gets wrong by five to fifteen percent and creates billing disputes that erode trust.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/metered-billing-agent) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** You charge customers a flat monthly fee but their usage varies wildly. Heavy users pay too little and you lose money. Light users pay too much and they churn. You know metered billing fixes this but the reconciliation logic terrifies you.

**After:** Every customer action emits a usage event that streams to Stripe in real-time. At the end of each billing period, Stripe generates an accurate invoice with no manual reconciliation. Heavy users pay for what they use. Light users stay because they pay only for what they need.

## Tools used

stripe, anthropic-api

## Pillar

[Payments](https://thebuilderweekly.com/tutorials/pillars/payments)
