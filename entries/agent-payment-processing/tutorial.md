## Opening thesis

You will build an AI agent that processes Stripe charges with automatic retries and idempotency keys. The agent decides when and how much to charge, then handles every failure mode, including network timeouts, 500 errors, and ambiguous responses, without human intervention. An agent retrying a charge without idempotency keys has charged the same customer 47 times before a human noticed; with proper idempotency the same agent can safely retry indefinitely.

## Before

Your agent calls the Stripe charges API. The request leaves your server. Stripe processes it. The response never arrives because a load balancer between you and Stripe drops the connection. Your agent sees a timeout. It retries. Stripe processes the second request as a new charge. The customer pays twice. You find out three days later when the chargeback notice hits your inbox. You refund, apologize, and lose the dispute fee anyway. Multiply this by every charge your agent makes during off-hours, weekends, and deploy windows when nobody is watching the logs. The agent is fast, persistent, and has no concept of "maybe I should wait and check." Without guard rails, that persistence is expensive.

## Architecture

The system has four components: a Claude-based agent that decides when to charge, a Python payment service that wraps Stripe calls with idempotency, a Stripe webhook listener that confirms charge outcomes, and a local SQLite ledger that tracks each logical payment's status. The agent never talks to Stripe directly. It calls the payment service, which enforces idempotency at the application layer before the request reaches Stripe.

```text
DIAGRAM: Agent Payment Flow
Caption: Shows how the agent triggers a charge through the payment service, with idempotency enforced at two layers.
Nodes:
1. Claude Agent - Decides charge amount, customer, and intent ID for each logical payment
2. Payment Service (Python) - Generates idempotency key from intent ID, calls Stripe, records result
3. Stripe API - Processes charges, enforces server-side idempotency for 24 hours
4. Webhook Listener - Receives charge.succeeded and charge.failed events from Stripe
5. SQLite Ledger - Stores intent ID, idempotency key, charge status, Stripe charge ID
Flow:
- Claude Agent sends (customer_id, amount, intent_id) to Payment Service
- Payment Service checks SQLite Ledger for existing intent_id
- If not found, Payment Service calls Stripe API with idempotency key derived from intent_id
- Stripe API returns charge result to Payment Service
- Payment Service writes result to SQLite Ledger
- Stripe API sends webhook event to Webhook Listener
- Webhook Listener updates SQLite Ledger with confirmed status
- Claude Agent queries SQLite Ledger to verify charge landed
```

## Step-by-step implementation

### 1. Install dependencies

You need the Stripe Python library, Flask for the webhook listener, and the Anthropic SDK for the agent. Install all three.

```bash
pip install stripe flask anthropic
```

### 2. Set environment variables

Get your Stripe secret key from https://dashboard.stripe.com/apikeys. Get your Anthropic API key from https://console.anthropic.com/settings/keys. Get your Stripe webhook signing secret from the webhooks section of the Stripe dashboard after you create the endpoint in step 7.

```bash
export STRIPE_SECRET_KEY="sk_test_..."
export STRIPE_WEBHOOK_SECRET="whsec_..."
export ANTHROPIC_API_KEY="sk-ant-..."
```

### 3. Create the SQLite ledger

The ledger tracks every logical payment intent your agent creates. The `intent_id` column is the primary key. This is the source of truth your agent checks before retrying.

```python
# ledger.py
import sqlite3

def init_db(path="payments.db"):
    conn = sqlite3.connect(path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS payments (
            intent_id TEXT PRIMARY KEY,
            idempotency_key TEXT NOT NULL,
            customer_id TEXT NOT NULL,
            amount_cents INTEGER NOT NULL,
            currency TEXT NOT NULL DEFAULT 'usd',
            stripe_charge_id TEXT,
            status TEXT NOT NULL DEFAULT 'pending',
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    return conn

def get_payment(conn, intent_id):
    row = conn.execute(
        "SELECT intent_id, stripe_charge_id, status FROM payments WHERE intent_id = ?",
        (intent_id,)
    ).fetchone()
    if row:
        return {"intent_id": row[0], "stripe_charge_id": row[1], "status": row[2]}
    return None

def upsert_payment(conn, intent_id, idempotency_key, customer_id, amount_cents, currency, stripe_charge_id, status):
    conn.execute("""
        INSERT INTO payments (intent_id, idempotency_key, customer_id, amount_cents, currency, stripe_charge_id, status, updated_at)
        VALUES (?, ?, ?, ?, ?, ?, ?, CURRENT_TIMESTAMP)
        ON CONFLICT(intent_id) DO UPDATE SET
            stripe_charge_id = excluded.stripe_charge_id,
            status = excluded.status,
            updated_at = CURRENT_TIMESTAMP
    """, (intent_id, idempotency_key, customer_id, amount_cents, currency, stripe_charge_id, status))
    conn.commit()
```

### 4. Build the payment service

This module wraps Stripe's charge creation. It derives the idempotency key deterministically from the intent ID. If the ledger already shows a succeeded charge for this intent, it short-circuits and returns the existing charge ID. If the ledger shows pending or failed, it retries with the same idempotency key.

```python
# payment_service.py
import os
import hashlib
import stripe
from ledger import init_db, get_payment, upsert_payment

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]

def make_idempotency_key(intent_id):
    return "agent_pay_" + hashlib.sha256(intent_id.encode()).hexdigest()[:32]

def charge_customer(intent_id, customer_id, amount_cents, currency="usd"):
    conn = init_db()
    existing = get_payment(conn, intent_id)

    if existing and existing["status"] == "succeeded":
        return {"already_charged": True, "charge_id": existing["stripe_charge_id"]}

    idem_key = make_idempotency_key(intent_id)

    upsert_payment(conn, intent_id, idem_key, customer_id, amount_cents, currency, None, "pending")

    try:
        charge = stripe.Charge.create(
            amount=amount_cents,
            currency=currency,
            customer=customer_id,
            description=f"Agent charge for intent {intent_id}",
            idempotency_key=idem_key
        )
        status = "succeeded" if charge.status == "succeeded" else "failed"
        upsert_payment(conn, intent_id, idem_key, customer_id, amount_cents, currency, charge.id, status)
        return {"already_charged": False, "charge_id": charge.id, "status": status}

    except stripe.error.StripeError as e:
        upsert_payment(conn, intent_id, idem_key, customer_id, amount_cents, currency, None, "error")
        raise e
```

### 5. Build the agent

The agent uses Claude to decide whether to charge a customer. It receives a task description, reasons about the charge, then calls `charge_customer`. If the call fails, the agent retries with the same `intent_id`. Because the idempotency key is derived from the intent ID, every retry is safe.

```python
# agent.py
import os
import json
import anthropic
from payment_service import charge_customer

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

TOOLS = [
    {
        "name": "process_charge",
        "description": "Charge a customer. Safe to retry: uses idempotency keys internally.",
        "input_schema": {
            "type": "object",
            "properties": {
                "intent_id": {"type": "string", "description": "Unique ID for this logical payment. Use the same ID for retries."},
                "customer_id": {"type": "string", "description": "Stripe customer ID (cus_...)"},
                "amount_cents": {"type": "integer", "description": "Amount in cents"},
                "currency": {"type": "string", "default": "usd"}
            },
            "required": ["intent_id", "customer_id", "amount_cents"]
        }
    }
]

def run_agent(task):
    messages = [{"role": "user", "content": task}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=TOOLS,
            messages=messages
        )

        if response.stop_reason == "tool_use":
            tool_block = next(b for b in response.content if b.type == "tool_use")
            args = tool_block.input
            try:
                result = charge_customer(
                    intent_id=args["intent_id"],
                    customer_id=args["customer_id"],
                    amount_cents=args["amount_cents"],
                    currency=args.get("currency", "usd")
                )
                tool_result = json.dumps(result)
            except Exception as e:
                tool_result = json.dumps({"error": str(e), "retryable": True})

            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": [{"type": "tool_result", "tool_use_id": tool_block.id, "content": tool_result}]})
        else:
            final_text = next((b.text for b in response.content if hasattr(b, "text")), "")
            return final_text

if __name__ == "__main__":
    result = run_agent("Charge customer cus_TEST123 exactly $49.99 for their monthly subscription. Use intent ID sub_2026_04_TEST123.")
    print(result)
```

### 6. Build the webhook listener

Webhooks are your confirmation layer. Even if the agent's HTTP call to Stripe times out, Stripe still sends the event. The webhook listener updates the ledger so the agent can verify the charge landed.

```python
# webhook_listener.py
import os
import json
from flask import Flask, request, abort
import stripe
from ledger import init_db, upsert_payment

app = Flask(__name__)
stripe.api_key = os.environ["STRIPE_SECRET_KEY"]
webhook_secret = os.environ["STRIPE_WEBHOOK_SECRET"]

@app.route("/webhook", methods=["POST"])
def handle_webhook():
    payload = request.get_data(as_text=True)
    sig_header = request.headers.get("Stripe-Signature")

    try:
        event = stripe.Webhook.construct_event(payload, sig_header, webhook_secret)
    except (ValueError, stripe.error.SignatureVerificationError):
        abort(400)

    if event["type"] in ("charge.succeeded", "charge.failed"):
        charge = event["data"]["object"]
        description = charge.get("description", "")
        if "intent" in description:
            intent_id = description.split("intent ")[-1]
            conn = init_db()
            status = "succeeded" if event["type"] == "charge.succeeded" else "failed"
            upsert_payment(
                conn, intent_id, "", charge["customer"],
                charge["amount"], charge["currency"],
                charge["id"], status
            )

    return json.dumps({"received": True}), 200

if __name__ == "__main__":
    app.run(port=4242)
```

### 7. Register the webhook with Stripe

Use the Stripe CLI to forward events to your local listener during development. In production, register the endpoint URL in the [Stripe Dashboard webhooks page](https://dashboard.stripe.com/webhooks).

```bash
stripe listen --forward-to localhost:4242/webhook
```

### 8. Test the retry behavior

Run the agent twice with the same intent ID. The second run should return `already_charged: true` without creating a new charge. This confirms both the application-layer and Stripe-layer idempotency are working.

```python
# test_retry.py
from payment_service import charge_customer

first = charge_customer("test_intent_001", "cus_TEST123", 4999)
print(f"First call: {first}")

second = charge_customer("test_intent_001", "cus_TEST123", 4999)
print(f"Second call: {second}")

assert second["already_charged"] is True
assert first["charge_id"] == second["charge_id"]
print("Idempotency verified: same charge ID, no duplicate.")
```

## Breakage

Remove the ledger check and the idempotency key from `charge_customer`. Now each call to `stripe.Charge.create` is a distinct charge. The agent hits a timeout, retries, and Stripe processes both requests. The customer sees two line items on their statement. Your agent, being diligent, retries on every ambiguous response. During a period of network instability lasting ten minutes, the agent fires 47 retries. Each one lands as a separate charge. You find out when the customer tweets about it.

```text
DIAGRAM: Breakage Without Idempotency
Caption: Shows duplicate charges created when the agent retries without idempotency keys.
Nodes:
1. Claude Agent - Retries on timeout, generates new charge each time
2. Stripe API - Treats each request as a new charge (no idempotency key)
3. Customer - Receives N charges for one logical payment
Flow:
- Agent sends charge request 1, gets timeout
- Agent sends charge request 2 (retry), Stripe creates charge 2
- Agent sends charge request 3 (retry), Stripe creates charge 3
- Customer is charged N times for one purchase
- Support team discovers duplicates days later via chargeback
```

## The fix

The fix is already built into the payment service from step 4. The key mechanism has two layers. First, the application layer: `get_payment` checks the ledger before calling Stripe. If the intent already succeeded, no HTTP request fires. Second, the Stripe layer: even if two requests escape the application check (race condition during concurrent retries), the `idempotency_key` parameter tells Stripe to return the original response instead of creating a new charge. Stripe caches idempotency results for 24 hours. Here is the critical section isolated:

```python
# The two-layer idempotency guard from payment_service.py

# Layer 1: Application-level check
existing = get_payment(conn, intent_id)
if existing and existing["status"] == "succeeded":
    return {"already_charged": True, "charge_id": existing["stripe_charge_id"]}

# Layer 2: Stripe-level idempotency key
idem_key = make_idempotency_key(intent_id)
charge = stripe.Charge.create(
    amount=amount_cents,
    currency=currency,
    customer=customer_id,
    description=f"Agent charge for intent {intent_id}",
    idempotency_key=idem_key
)
```

## Fixed state

```text
DIAGRAM: Agent Payment Flow With Two-Layer Idempotency
Caption: Shows how retries are absorbed at both the application and Stripe layers.
Nodes:
1. Claude Agent - Retries freely with the same intent_id
2. Payment Service - Checks ledger first, attaches idempotency key to every Stripe call
3. SQLite Ledger - Returns existing charge status, preventing redundant API calls
4. Stripe API - Returns cached result for duplicate idempotency keys within 24 hours
5. Webhook Listener - Confirms final charge status asynchronously
Flow:
- Agent sends (intent_id, customer_id, amount) to Payment Service
- Payment Service queries SQLite Ledger for intent_id
- If found with status succeeded, Payment Service returns existing charge_id (no Stripe call)
- If not found or status is error, Payment Service calls Stripe API with idempotency_key
- Stripe API returns original charge result if idempotency_key was seen before
- Payment Service writes result to SQLite Ledger
- Webhook Listener receives event, updates Ledger as async confirmation
- Agent reads Ledger to confirm charge landed exactly once
```

## After

Your agent calls the payment service. The request leaves your server. Stripe processes it. The response never arrives. Your agent sees a timeout. It retries with the same intent ID. The payment service checks the ledger, finds no confirmed charge, and calls Stripe with the same idempotency key. Stripe recognizes the key and returns the original charge result. One charge. One line item on the customer's statement. The agent retries twelve more times during a network flap. Every retry resolves to the same charge. No chargebacks. No apology emails. No one tweets about it. You find out about the incident only because your observability dashboard shows a spike in retries, with zero duplicate charges.

## Takeaway

The pattern is deterministic request identity. Every operation your agent performs against an external system needs a stable identifier that survives retries. Derive that identifier from the business intent, not from the request itself. This applies to charges, to email sends, to database writes, to any side effect. If your agent can retry, and it will, the external system must be able to recognize the retry as a duplicate. Build that recognition into every integration your agent touches.