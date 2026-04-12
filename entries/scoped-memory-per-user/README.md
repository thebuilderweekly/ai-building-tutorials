# Scope agent memory per user without leaking across sessions

An agent can index and retrieve against millions of per-user memory stores in parallel, which a human operating the same product would need a team and a custom dashboard to match.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/scoped-memory-per-user) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agent's memory is a single global store and one user's context bleeds into another user's responses.

**After:** Every memory write is namespaced to a user id, and retrieval only sees the requesting user's own history.

## Tools used

mem0

## Pillar

[Memory](https://thebuilderweekly.com/tutorials/pillars/memory)
