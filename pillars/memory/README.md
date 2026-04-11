# Memory

Persistent state across agent sessions. Tutorials in this pillar teach patterns for writing, retrieving, and scoping agent memory so the agent gets better with every interaction instead of starting from zero every time.

## Entries

_No entries yet. The first memory tutorial is in the queue — see [queue/topics.json](../../queue/topics.json)._

## The pattern

Every memory tutorial follows the same arc: an agent writes something it learned, the write is scoped to the right owner (user, session, tenant, project), and a later retrieval returns the right fact at the right time.

- **Before-state:** "The agent forgets everything when the session ends."
- **After-state:** "The agent retrieves relevant context from previous sessions before responding, and gets better with every interaction."

The load-bearing question in every memory tutorial is "what's the scope key, and how does retrieval find the right chunk?" A tutorial that teaches a global memory store with no scoping isn't finished — that's just a database.

## Cluster tags in this pillar

- `mem0` — entries that use Mem0 as the memory layer
- `persistence` — entries centered on the write side
- `context-retrieval` — entries centered on the read side
- `isolation` — entries for multi-tenant scoping
- `multi-tenant` — entries for cross-user isolation patterns
- `retrieval` — entries for the retrieval query itself
- `vector-stores` — entries that use a vector store directly
- `metadata-filters` — entries that combine semantic and metadata queries
