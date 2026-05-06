# Retrieve from a vector store with metadata filters and get the right chunk

An agent combining semantic search with metadata filters reaches into ten million chunks and returns the right one in under a hundred milliseconds, which a human searching a SQL database by hand cannot come close to matching.

This is part of [AI Building Tutorials](https://github.com/thebuilderweekly/ai-building-tutorials) by [The Builder Weekly](https://thebuilderweekly.com).

**Read this tutorial:**
- [In this repo](./tutorial.md) — the raw markdown with code blocks
- [On the web](https://thebuilderweekly.com/tutorials/retrieval-with-metadata-filters) — rendered with diagrams and syntax highlighting

## What this tutorial teaches

**Before:** Your agent's retrieval returns ten loosely relevant chunks because pure semantic search can't tell which belong to which user or which date range.

**After:** The same retrieval with metadata filters returns one chunk, it is the right one, and the latency is identical.

## Tools used

qdrant, anthropic-api

## Pillar

[Memory](https://thebuilderweekly.com/tutorials/pillars/memory)
