## Opening thesis

You will build a metered billing system using Stripe's usage records API. Every action your agent performs on a customer's behalf emits a usage event that streams to Stripe in real time. At the end of each billing period, Stripe generates an accurate invoice based on the recorded usage. An agent meters a thousand customer API calls per second with sub-cent precision, which a manual usage report at month-end gets wrong by five to fifteen percent and creates billing disputes that erode trust. By the end, your customers are billed exactly for what they used, automatically.

## Before

You ship a flat-rate SaaS plan: $99 per month for unlimited use. Two months in, you notice the usage distribution: 80% of customers use less than 1% of your infrastructure capacity. The other 20% use 99% of it. Heavy users are subsidized by light users. Light users churn because they feel overcharged. Heavy users churn when they grow because the next plan tier is twice the price for marginal additional value. You decide to add metered billing. You build a usage tracker. It writes to a Postgres table. At month-end, a script reads the table, computes per-customer totals, and sends them to your billing system. The first month, the script crashes halfway through. You patch it. The second month, a customer disputes the bill because your tracker double-counted retries. You issue refunds. The third month, you spot-check ten random invoices and find three discrepancies. You stop trusting the system. You go back to flat pricing. You leave money on the table and accept the churn.

## Architecture

The system has four components: a usage event emitter that runs inside your agent, a Stripe customer with a metered subscription product, a usage records writer that streams events to Stripe, and a webhook handler that confirms billing events. Stripe is the source of truth for usage. Your local database is for read-only auditing.

```text
DIAGRAM: Metered billing agent system
Caption: Every customer action streams a usage event to Stripe, which produces accurate invoices automatically
Nodes:
1. Customer action - User performs a billable operation (API call, agent run, document processed)
2. Agent runtime - Executes the action and triggers a usage event
3. Usage event emitter - Constructs a structured event with customer ID, quantity, and timestamp
4. Stripe usage records API - Receives the event, attaches it to the customer's subscription item
5. Stripe billing engine - Aggregates usage records over the billing period
6. Local audit log - Records every emitted event for read-only customer-facing reports
7. Stripe webhook handler - Receives invoice.created, invoice.paid, and invoice.payment_failed events
8. Customer dashboard - Shows real-time usage from local audit log, plus billing history from Stripe
Flow:
- Customer action triggers Agent runtime to perform the operation
- Agent runtime emits a usage event via Usage event emitter
- Usage event emitter sends the event to Stripe usage records API
- Stripe usage records API attaches the record to the customer's subscription item
- Usage event emitter also writes the event to Local audit log
- At billing period end: Stripe billing engine aggregates usage records into an invoice
- Stripe webhook handler receives invoice events and updates customer status
- Customer dashboard reads usage from Local audit log and billing from Stripe
```

## Step-by-step implementation

### Step 1: Install dependencies

You need the Stripe Python SDK, Flask for webhook handling, and the Anthropic SDK if your billable actions involve LLM calls.

```bash
pip install stripe flask anthropic
```

### Step 2: Set environment variables

Get your Stripe keys from https://dashboard.stripe.com/apikeys. Get the webhook signing secret after creating the webhook endpoint in Step 7. Get your Anthropic key from https://console.anthropic.com/settings/keys.

```bash
export STRIPE_SECRET_KEY="sk_test_..."
export STRIPE_WEBHOOK_SECRET="whsec_..."
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Step 3: Create a metered product and price in Stripe

Use the Stripe dashboard or the API to create a product with a metered price. Each unit of usage will be billed at the per-unit rate. This script creates the product programmatically.

```python
# create_product.py
import os
import stripe

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]

product = stripe.Product.create(
    name="Agent Operations",
    description="Per-operation billing for agent runs",
)

price = stripe.Price.create(
    product=product.id,
    currency="usd",
    recurring={"interval": "month", "usage_type": "metered"},
    billing_scheme="per_unit",
    unit_amount=2,  # $0.02 per operation, in cents
)

print(f"Created product: {product.id}")
print(f"Created metered price: {price.id}")
print(f"Save the price ID; you will need it to create subscriptions.")
```

Save the printed `price.id` value; you will reference it as `METERED_PRICE_ID` below.

### Step 4: Create a customer and subscription

When a new customer signs up, create a Stripe customer and attach a subscription using the metered price. Store the resulting subscription item ID; usage records reference this ID, not the subscription ID.

```python
# customer_setup.py
import os
import stripe

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]

METERED_PRICE_ID = "price_..."  # paste from Step 3

def create_metered_customer(email: str, payment_method: str) -> dict:
    customer = stripe.Customer.create(
        email=email,
        payment_method=payment_method,
        invoice_settings={"default_payment_method": payment_method},
    )

    subscription = stripe.Subscription.create(
        customer=customer.id,
        items=[{"price": METERED_PRICE_ID}],
        collection_method="charge_automatically",
    )

    subscription_item_id = subscription["items"]["data"][0]["id"]

    return {
        "customer_id": customer.id,
        "subscription_id": subscription.id,
        "subscription_item_id": subscription_item_id,
    }
```

### Step 5: Build the usage event emitter

This is the core. Every billable action calls `emit_usage`, which posts a usage record to Stripe. The function takes the subscription item ID and a quantity (usually 1, but can be higher for batch operations).

```python
# usage_emitter.py
import os
import time
import sqlite3
import stripe

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]

def init_audit_db(path="usage_audit.db"):
    conn = sqlite3.connect(path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS usage_events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            customer_id TEXT NOT NULL,
            subscription_item_id TEXT NOT NULL,
            quantity INTEGER NOT NULL,
            action_type TEXT NOT NULL,
            stripe_record_id TEXT,
            timestamp TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    return conn

def emit_usage(
    customer_id: str,
    subscription_item_id: str,
    quantity: int,
    action_type: str,
) -> dict:
    conn = init_audit_db()

    timestamp = int(time.time())

    record = stripe.SubscriptionItem.create_usage_record(
        subscription_item_id,
        quantity=quantity,
        timestamp=timestamp,
        action="increment",
    )

    conn.execute(
        "INSERT INTO usage_events (customer_id, subscription_item_id, quantity, action_type, stripe_record_id) VALUES (?, ?, ?, ?, ?)",
        (customer_id, subscription_item_id, quantity, action_type, record.id),
    )
    conn.commit()
    conn.close()

    return {"stripe_record_id": record.id, "quantity": quantity}
```

The `action="increment"` parameter ensures duplicate calls are additive rather than overwriting. The `timestamp` parameter records when the action happened, which Stripe uses for invoice line item dates.

### Step 6: Wire usage emission into your agent

Wherever your agent performs a billable action, call `emit_usage`. This example shows a simple agent that processes a document and emits one usage unit per processing run.

```python
# agent.py
import os
import anthropic
from usage_emitter import emit_usage

client = anthropic.Anthropic()

def process_document(
    customer_id: str,
    subscription_item_id: str,
    document_text: str,
) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=512,
        messages=[
            {"role": "user", "content": f"Summarize this document in 100 words: {document_text}"}
        ],
    )

    summary = response.content[0].text

    emit_usage(
        customer_id=customer_id,
        subscription_item_id=subscription_item_id,
        quantity=1,
        action_type="document_processed",
    )

    return {"summary": summary}
```

### Step 7: Build the webhook handler

Stripe sends webhook events when invoices are created, paid, or fail. The handler logs these events and triggers downstream actions like sending a receipt email or pausing service for non-payment.

```python
# webhook_handler.py
import os
from flask import Flask, request, jsonify
import stripe

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
        return jsonify({"error": "invalid signature"}), 400

    event_type = event["type"]
    obj = event["data"]["object"]

    if event_type == "invoice.created":
        print(f"Invoice created for customer {obj['customer']}: {obj['amount_due'] / 100} {obj['currency']}")
    elif event_type == "invoice.paid":
        print(f"Invoice paid: {obj['id']}")
    elif event_type == "invoice.payment_failed":
        print(f"Payment failed for invoice {obj['id']}")

    return jsonify({"received": True}), 200

if __name__ == "__main__":
    app.run(port=4242)
```

Register the webhook endpoint in the Stripe dashboard at https://dashboard.stripe.com/webhooks. Subscribe to `invoice.created`, `invoice.paid`, and `invoice.payment_failed`.

### Step 8: Build a real-time usage report for the customer dashboard

Customers want to see their usage as it accrues, not wait for the invoice. This function reads from the local audit log and returns the customer's current-period usage.

```python
# usage_report.py
import sqlite3
from datetime import datetime, timedelta

def get_current_period_usage(customer_id: str, period_days: int = 30) -> dict:
    conn = sqlite3.connect("usage_audit.db")
    cutoff = (datetime.utcnow() - timedelta(days=period_days)).isoformat()

    rows = conn.execute(
        "SELECT action_type, SUM(quantity) FROM usage_events WHERE customer_id = ? AND timestamp >= ? GROUP BY action_type",
        (customer_id, cutoff),
    ).fetchall()

    by_action = {action: total for action, total in rows}
    total_units = sum(by_action.values())

    conn.close()

    return {
        "customer_id": customer_id,
        "period_days": period_days,
        "total_units": total_units,
        "by_action_type": by_action,
        "estimated_cost_usd": total_units * 0.02,
    }
```

### Step 9: Test end-to-end

Create a test customer, run a few billable actions, and verify the usage records appear in Stripe.

```python
# test_metered.py
from customer_setup import create_metered_customer
from agent import process_document
from usage_report import get_current_period_usage

customer = create_metered_customer(
    email="test@example.com",
    payment_method="pm_card_visa",  # Stripe test payment method
)

print(f"Created customer: {customer['customer_id']}")

for i in range(5):
    result = process_document(
        customer_id=customer["customer_id"],
        subscription_item_id=customer["subscription_item_id"],
        document_text=f"Test document number {i}.",
    )
    print(f"Processed document {i}")

report = get_current_period_usage(customer["customer_id"])
print(f"Current usage: {report}")
```

Check the Stripe dashboard under the customer's subscription. You should see five usage records and an estimated invoice total of $0.10.

## Breakage

Skip the streaming emission. Instead, write usage to a local table and run a nightly batch job that computes totals and pushes them to Stripe. The first night, the batch job crashes after processing half the customers. The other half have no usage recorded for that day. The next night, your patch reruns the missed customers but double-counts the ones that succeeded the first time. Customers receive invoices with strange line items. Some get billed twice. Some not at all. Disputes pile up. You spend a week reconciling. You rebuild the batch job to be idempotent. The next month a network blip causes a partial sync. The cycle repeats.

```text
DIAGRAM: Batch reconciliation failure mode
Caption: Nightly reconciliation produces drift, duplicates, and disputes
Nodes:
1. Agent runtime - Performs billable actions
2. Local usage table - Stores raw events
3. Nightly batch job - Aggregates and pushes to Stripe
4. Failure point - Job crashes mid-run
5. Manual reconciliation - Engineer must figure out what synced and what did not
6. Customer dispute - Invoice is wrong, customer complains
Flow:
- Agent runtime writes usage events to Local usage table
- Nightly batch job runs at 2 AM
- Job crashes after processing 50% of customers
- Half the customers have correct invoices, half have nothing
- Engineer runs reconciliation by hand
- Some customers get double-billed by the patch script
- Customer dispute reaches support inbox
```

## The fix

The fix is the streaming emission from Step 5. Every billable action emits a usage record to Stripe synchronously, in the same code path as the action itself. There is no batch job. There is no reconciliation. The critical pattern is the immediate emission inside the agent runtime, isolated below.

```python
# The streaming emission pattern from agent.py
def process_document(customer_id, subscription_item_id, document_text):
    # Step 1: perform the billable action
    response = client.messages.create(...)
    summary = response.content[0].text

    # Step 2: emit usage immediately, in the same call
    # If this fails, the action is rolled back via exception propagation
    emit_usage(
        customer_id=customer_id,
        subscription_item_id=subscription_item_id,
        quantity=1,
        action_type="document_processed",
    )

    return {"summary": summary}
```

The action and the usage record are coupled in the same function call. If the usage emission raises an exception, your application code can decide whether to retry or abort the action. Stripe's `action="increment"` parameter ensures retries are additive without duplication. The audit log is for read-only customer-facing reports, not for billing reconciliation. Stripe is the source of truth.

## Fixed state

```text
DIAGRAM: Streaming metered billing system
Caption: Every billable action streams immediately to Stripe with a local audit copy
Nodes:
1. Customer action - User performs a billable operation
2. Agent runtime - Performs the action
3. Usage event emitter - Sends usage record to Stripe synchronously
4. Stripe usage records API - Receives and attaches to subscription item
5. Local audit log - Records the same event for customer-facing reports
6. Stripe billing engine - Aggregates usage at billing period end
7. Stripe webhook handler - Receives invoice events and updates customer state
8. Customer dashboard - Reads real-time usage from local log and billing from Stripe
Flow:
- Customer action triggers Agent runtime
- Agent runtime performs the action
- Usage event emitter posts to Stripe usage records API and writes to Local audit log
- Stripe usage records API attaches the record to the customer subscription
- Stripe billing engine aggregates records continuously
- At billing period end, Stripe generates an invoice from accumulated records
- Stripe webhook handler receives invoice events
- Customer dashboard shows usage in near-real-time
```

## After

A customer triggers a thousand agent operations in an afternoon. Each one streams a usage record to Stripe within milliseconds. Their dashboard updates in near-real-time, showing 1,000 operations at $0.02 each, a running total of $20.00 for the period. At the end of the billing period, Stripe generates an invoice for $20.00. The customer pays. The webhook fires. Your handler logs the payment. Zero reconciliation. Zero disputes. Heavy users pay for what they use. Light users pay only for what they need and stop churning. Your monthly recurring revenue stops fluctuating wildly because heavy users no longer subsidize light ones, and the relationship between value delivered and revenue captured is transparent to both sides.

## Takeaway

The pattern is synchronous emission at the point of action. Every billable event flows to the billing system in the same code path that performs the action. There is no batch, no reconciliation, no nightly job. The billing system becomes the source of truth, and your local audit log is for read-only purposes only. Apply this anywhere precise per-event accounting matters: API rate billing, marketplace transactions, ad impressions, content licensing, compute usage. Streaming beats batch every time when accuracy matters and disputes are expensive.
