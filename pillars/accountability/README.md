# Accountability

Proving agent work actually happened. Tutorials in this pillar teach patterns for scheduling, verifying, and auditing agent outputs so you know the work was done, not just claimed.

## Entries

_No entries yet. The first accountability tutorial is in the queue — see [queue/topics.json](../../queue/topics.json)._

## The pattern

Every accountability tutorial follows the same arc: an agent does work, something verifies the work actually happened, and the verification evidence is stored where it can be audited later.

- **Before-state:** "The agent says it did the work."
- **After-state:** "You have timestamped, externally-verifiable proof the work happened."

The load-bearing artifact in every accountability tutorial is the verification evidence. A tutorial that shows you how to check an outcome without showing you where the evidence lives, or how to retrieve it six months later, isn't finished.

## Cluster tags in this pillar

- `cueapi` — entries that use CueAPI as the accountability substrate
- `verification` — entries centered on the verification step itself
- `outcome-tracking` — entries focused on recording what actually happened
- `cron` — entries for scheduled job accountability
- `drift-detection` — entries for reconciling two systems of record
- `job-verification` — entries that wrap individual jobs in proof
