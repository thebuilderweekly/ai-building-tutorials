# Visibility

Making agent-built sites visible to crawlers. Tutorials in this pillar teach patterns for exposing content so that AI crawlers and retrieval systems can actually read it — not just render it on a screen for a human.

## Entries

_No entries yet. The first visibility tutorial is in the queue — see [queue/topics.json](../../queue/topics.json)._

## The pattern

Every visibility tutorial follows the same arc: a site has content, a crawler tries to read it, the crawler fails on some specific dimension (JavaScript-rendered, unstructured HTML, no corpus index), the tutorial shows how to fix that specific dimension, and the fix is verified by pointing a real crawler at the output.

- **Before-state:** "The crawler sees an empty page (or unparseable HTML, or a sprawling site with no guide)."
- **After-state:** "The crawler reads the content cleanly, extracts structured fields, and indexes the pages that matter in the order you chose."

The load-bearing verification in every visibility tutorial is an actual crawler check. A tutorial that says "now add JSON-LD" and doesn't show what a crawler sees after the change isn't finished — the whole point is to prove the before/after transformation is real.

## Cluster tags in this pillar

- `llms-txt` — entries centered on the llms.txt contract
- `crawlers` — entries focused on crawler behavior
- `agent-readable` — entries about making pages agent-parseable
- `ssr` — entries about server-side rendering for crawler visibility
- `javascript-rendering` — entries about CSR-to-static-snapshot patterns
- `json-ld` — entries about schema.org JSON-LD embedding
- `structured-data` — entries about structured metadata in any form
- `schema-org` — entries that use schema.org vocabularies
