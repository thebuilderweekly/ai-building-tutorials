# Payments

Patterns for agents handling money. Charges, subscriptions, refunds, payouts. Tutorials in this pillar teach idempotency, webhook verification, and the failure modes that matter when dollars are at stake.

## Entries

_No entries yet. The first payments tutorial is in the queue._

## The pattern

Every payments tutorial follows the same arc: an agent performs a financial action, something can go wrong that costs real money (double charges, missed refunds, stale webhooks), and the tutorial shows the exact code that prevents the loss.

- **Before-state:** "The agent charges customers and sometimes charges them twice."
- **After-state:** "The agent charges customers exactly once, with proof."

## Cluster tags in this pillar

- `stripe` — entries using Stripe for payment processing
- `idempotency` — entries focused on preventing duplicate operations
- `webhooks` — entries for processing asynchronous payment confirmations
