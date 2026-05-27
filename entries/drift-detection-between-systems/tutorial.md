## Opening thesis

You will build a reconciliation agent that watches writes to a billing system and a CRM, compares the records in both systems within seconds, and logs any mismatch with full state from both sides. An agent reconciles two systems of record on every write with zero fatigue, catching silent divergence within a minute, where a human running a quarterly reconciliation script finds the same drift weeks after it happened. The agent uses Cue to orchestrate the reconciliation workflow and Claude to interpret ambiguous field mappings between the two systems.

## Before

Your billing system says the customer is on the Enterprise plan at $4,800 per year. Your CRM says they are on Professional at $2,400 per year. Nobody knows which is right because the records diverged three months ago when someone updated billing directly and the webhook to the CRM silently failed. You discover this during the quarterly audit. You open a spreadsheet. You pull every customer record from both systems. You diff 11,000 rows. You find 340 mismatches. For each one, you check timestamps, API logs, and Slack threads to figure out which system has the correct state. This takes a week. During that week, invoices go out with wrong amounts. Support tickets pile up. The root cause was a single dropped webhook. The cost was a week of engineering time and a handful of angry customers.

## Architecture

The system has four components. A webhook listener receives write events from both the billing system and the CRM. It sends each event to a Cue workflow. The workflow fetches the corresponding record from the other system, then calls Claude to compare the two records and determine if they match. If they do not match, the workflow logs a drift event to a structured log store with both records attached.

```text
DIAGRAM: Reconciliation agent architecture
Caption: Shows how every write triggers a cross-system comparison within seconds.
Nodes:
1. Billing System - source of subscription and invoice writes
2. CRM System - source of customer profile and plan writes
3. Webhook Listener (Node.js) - receives POST events from both systems
4. Cue Workflow - orchestrates fetch, compare, and log steps
5. Claude (Anthropic API) - compares two records, identifies mismatches
6. Drift Log (structured JSON file or database) - stores mismatch events with both sides
Flow:
- Billing System sends write event to Webhook Listener
- CRM System sends write event to Webhook Listener
- Webhook Listener triggers Cue Workflow with event payload
- Cue Workflow fetches counterpart record from the other system
- Cue Workflow sends both records to Claude for comparison
- Claude returns match/mismatch verdict with field-level diff
- Cue Workflow writes drift event to Drift Log if mismatch detected
```

## Step-by-step implementation

### 1. Set up the project and install dependencies

Create a new Node.js project. Install the Cue SDK for workflow orchestration and the Anthropic SDK for calling Claude. You need two environment variables: your Anthropic API key (get it at https://console.anthropic.com/settings/keys) and your Cue API key (get it at https://console.cue.dev/settings/api-keys).

```bash
mkdir drift-agent && cd drift-agent
npm init -y
npm install @anthropic-ai/sdk express
export ANTHROPIC_API_KEY="your-key-from-console"
export CUEAPI_API_KEY="your-key-from-cue-console"
```

### 2. Define the record schema for both systems

Create a shared schema file that describes the fields you expect in both systems. This is the contract the agent enforces. When either system writes a record, the agent maps it to this schema before comparing.

```ts
// schema.ts
export interface NormalizedRecord {
  customerId: string;
  email: string;
  planName: string;
  planPriceAnnual: number;
  status: "active" | "canceled" | "past_due";
  lastUpdated: string; // ISO 8601
  sourceSystem: "billing" | "crm";
}

export interface DriftEvent {
  detectedAt: string;
  customerId: string;
  billingRecord: NormalizedRecord | null;
  crmRecord: NormalizedRecord | null;
  mismatchedFields: string[];
  verdict: string;
}
```

### 3. Build the webhook listener

This Express server receives POST requests from both systems. Each request includes a customer ID and the updated fields. The listener normalizes the incoming payload and passes it to the reconciliation function.

```ts
// server.ts
import express from "express";
import { reconcile } from "./reconcile";

const app = express();
app.use(express.json());

app.post("/webhook/billing", async (req, res) => {
  const event = { source: "billing" as const, payload: req.body };
  const result = await reconcile(event);
  res.json(result);
});

app.post("/webhook/crm", async (req, res) => {
  const event = { source: "crm" as const, payload: req.body };
  const result = await reconcile(event);
  res.json(result);
});

app.listen(3100, () => {
  console.log("Drift agent listening on port 3100");
});
```

### 4. Fetch the counterpart record

When billing writes, you fetch the same customer from the CRM, and vice versa. This function simulates those fetches. In production, replace these with real API calls to your billing provider (Stripe, Chargebee, etc.) and your CRM (Salesforce, HubSpot, etc.).

```ts
// fetch-counterpart.ts
import { NormalizedRecord } from "./schema";

const BILLING_API = process.env.BILLING_API_URL || "http://localhost:3200/api/billing";
const CRM_API = process.env.CRM_API_URL || "http://localhost:3200/api/crm";

export async function fetchCounterpart(
  customerId: string,
  sourceSystem: "billing" | "crm"
): Promise<NormalizedRecord | null> {
  const targetUrl = sourceSystem === "billing"
    ? `${CRM_API}/customers/${customerId}`
    : `${BILLING_API}/customers/${customerId}`;

  const resp = await fetch(targetUrl);
  if (!resp.ok) return null;
  const data = await resp.json();
  return data as NormalizedRecord;
}
```

### 5. Compare records with Claude

Send both records to Claude. Ask it to list mismatched fields and provide a one-sentence verdict. Claude handles edge cases a regex never will: field name differences, currency formatting, timezone offsets in timestamps, and plan name synonyms ("Enterprise" vs "enterprise_annual").

```ts
// compare.ts
import Anthropic from "@anthropic-ai/sdk";
import { NormalizedRecord } from "./schema";

const client = new Anthropic();

export async function compareRecords(
  billing: NormalizedRecord,
  crm: NormalizedRecord
): Promise<{ mismatched: string[]; verdict: string }> {
  const prompt = `You are a data reconciliation agent. Compare these two records for the same customer and return a JSON object with two fields:
- "mismatched": an array of field names that differ between the two records
- "verdict": a one-sentence summary of the drift

If the records match on all fields, return {"mismatched": [], "verdict": "Records match."}.

Billing record:
${JSON.stringify(billing, null, 2)}

CRM record:
${JSON.stringify(crm, null, 2)}

Return only valid JSON. No explanation.`;

  const message = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 512,
    messages: [{ role: "user", content: prompt }],
  });

  const text = message.content[0].type === "text" ? message.content[0].text : "{}";
  return JSON.parse(text);
}
```

### 6. Wire up the reconciliation function

This is the core. It receives the write event, fetches the counterpart, compares, and decides whether to log a drift event.

```ts
// reconcile.ts
import { fetchCounterpart } from "./fetch-counterpart";
import { compareRecords } from "./compare";
import { logDrift } from "./drift-log";
import { NormalizedRecord, DriftEvent } from "./schema";

interface WriteEvent {
  source: "billing" | "crm";
  payload: NormalizedRecord;
}

export async function reconcile(event: WriteEvent) {
  const { source, payload } = event;
  const counterpart = await fetchCounterpart(payload.customerId, source);

  if (!counterpart) {
    return { status: "skipped", reason: "counterpart not found" };
  }

  const billing = source === "billing" ? payload : counterpart;
  const crm = source === "crm" ? payload : counterpart;

  const result = await compareRecords(billing, crm);

  if (result.mismatched.length > 0) {
    const driftEvent: DriftEvent = {
      detectedAt: new Date().toISOString(),
      customerId: payload.customerId,
      billingRecord: billing,
      crmRecord: crm,
      mismatchedFields: result.mismatched,
      verdict: result.verdict,
    };
    await logDrift(driftEvent);
    return { status: "drift_detected", drift: driftEvent };
  }

  return { status: "in_sync" };
}
```

### 7. Implement the drift log

Write drift events to a local JSON Lines file. In production, send these to your observability platform, a database, or a Slack channel. The important thing: both sides of the record are captured at detection time, not hours later when someone investigates.

```ts
// drift-log.ts
import { appendFileSync } from "fs";
import { DriftEvent } from "./schema";

const LOG_PATH = process.env.DRIFT_LOG_PATH || "./drift-events.jsonl";

export async function logDrift(event: DriftEvent): Promise<void> {
  const line = JSON.stringify(event) + "\n";
  appendFileSync(LOG_PATH, line, "utf-8");
  console.log(
    `DRIFT: customer=${event.customerId} fields=${event.mismatchedFields.join(",")} verdict=${event.verdict}`
  );
}
```

### 8. Register the workflow in Cue

Use Cue to schedule and monitor the reconciliation agent. Cue tracks execution history, retries on transient failures, and gives you a dashboard to see how many drift events fired in the last hour. Register the workflow via the Cue API. See https://docs.cue.dev for the full API reference.

```bash
curl -X POST https://api.cue.dev/v1/workflows \
  -H "Authorization: Bearer $CUEAPI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "billing-crm-reconciliation",
    "trigger": "webhook",
    "endpoint": "https://your-server.example.com/webhook/billing",
    "retryPolicy": {
      "maxAttempts": 3,
      "backoffMs": 2000
    },
    "alertOnFailure": true
  }'
```

### 9. Test with a simulated drift event

Send a billing update where the plan name differs from what the CRM has. You should see a drift event logged within seconds.

```bash
curl -X POST http://localhost:3100/webhook/billing \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "cust_8832",
    "email": "ops@acme.co",
    "planName": "Enterprise",
    "planPriceAnnual": 4800,
    "status": "active",
    "lastUpdated": "2026-05-27T14:32:00Z",
    "sourceSystem": "billing"
  }'
```

### 10. Verify the drift log output

Check the log file. You should see a single JSON line with both the billing and CRM records, the mismatched fields, and Claude's verdict.

```bash
cat drift-events.jsonl | jq .
```

## Breakage

If you skip the per-write reconciliation step and rely on a periodic batch job, drift accumulates silently. A webhook fails on a Tuesday. Nobody notices. Three weeks pass. Invoices go out at the wrong price. A customer complains. Support opens a ticket. Engineering queries both systems, finds the mismatch, and then has to determine which system drifted. The timestamp trail is cold. Audit logs have been rotated. You spend two days reconstructing the sequence of events for one customer. Multiply that by every mismatched record in the quarterly audit.

```text
DIAGRAM: Failure mode without per-write reconciliation
Caption: Shows how a dropped webhook causes silent drift that compounds over weeks.
Nodes:
1. Billing System - writes a plan change
2. Webhook (to CRM) - fails silently, no retry
3. CRM System - retains stale data
4. Quarterly Audit Script - runs 90 days later
5. Engineering Team - manually investigates each mismatch
Flow:
- Billing System fires webhook to CRM System
- Webhook fails, CRM System never receives the update
- 90 days pass with no detection
- Quarterly Audit Script diffs all records, finds 340 mismatches
- Engineering Team spends a week resolving each mismatch manually
```

## The fix

The fix is the reconciliation call you already built in Step 6. The key addition is the Cue workflow retry policy from Step 8. If the counterpart system is temporarily unreachable, Cue retries the reconciliation up to three times with exponential backoff. If all retries fail, Cue fires an alert so a human can investigate. The drift log from Step 7 captures both records at the moment of detection, so there is no need to reconstruct state later. Here is the retry-aware version of the reconcile function that integrates with Cue's failure tracking.

```ts
// reconcile-with-retry.ts
import { fetchCounterpart } from "./fetch-counterpart";
import { compareRecords } from "./compare";
import { logDrift } from "./drift-log";
import { NormalizedRecord, DriftEvent } from "./schema";

interface WriteEvent {
  source: "billing" | "crm";
  payload: NormalizedRecord;
}

export async function reconcile(event: WriteEvent) {
  const { source, payload } = event;

  let counterpart: NormalizedRecord | null = null;
  try {
    counterpart = await fetchCounterpart(payload.customerId, source);
  } catch (err) {
    // Throw so Cue's retry policy catches it
    throw new Error(
      `Counterpart fetch failed for ${payload.customerId}: ${err}`
    );
  }

  if (!counterpart) {
    const orphanEvent: DriftEvent = {
      detectedAt: new Date().toISOString(),
      customerId: payload.customerId,
      billingRecord: source === "billing" ? payload : null,
      crmRecord: source === "crm" ? payload : null,
      mismatchedFields: ["existence"],
      verdict: `Record exists in ${source} but not in the other system.`,
    };
    await logDrift(orphanEvent);
    return { status: "orphan_detected", drift: orphanEvent };
  }

  const billing = source === "billing" ? payload : counterpart;
  const crm = source === "crm" ? payload : counterpart;
  const result = await compareRecords(billing, crm);

  if (result.mismatched.length > 0) {
    const driftEvent: DriftEvent = {
      detectedAt: new Date().toISOString(),
      customerId: payload.customerId,
      billingRecord: billing,
      crmRecord: crm,
      mismatchedFields: result.mismatched,
      verdict: result.verdict,
    };
    await logDrift(driftEvent);
    return { status: "drift_detected", drift: driftEvent };
  }

  return { status: "in_sync" };
}
```

## Fixed state

```text
DIAGRAM: Reconciliation agent with retry and orphan detection
Caption: Shows the complete system where every write triggers comparison, retries handle transient failures, and orphan records are flagged.
Nodes:
1. Billing System - source of subscription writes
2. CRM System - source of customer profile writes
3. Webhook Listener - receives events from both systems
4. Cue Workflow (with retry) - orchestrates reconciliation, retries on failure
5. Claude (Anthropic API) - compares records, handles field ambiguity
6. Drift Log - stores mismatch events with full state from both sides
7. Alert Channel - notified when retries exhausted
Flow:
- Billing System or CRM System sends write event to Webhook Listener
- Webhook Listener triggers Cue Workflow
- Cue Workflow fetches counterpart record from the other system
- If fetch fails, Cue retries up to 3 times with backoff
- If retries exhausted, Cue sends alert to Alert Channel
- If fetch succeeds, Cue sends both records to Claude
- Claude returns field-level diff
- If mismatch found, Cue writes drift event to Drift Log
- If record missing in other system, Cue logs orphan event to Drift Log
```

## After

Your billing system writes a plan change. Within 40 seconds, the agent fetches the CRM record, compares both, and logs a drift event with the exact fields that differ and the exact values on each side. You get an alert. You look at the drift log entry. It shows billing has "Enterprise" at $4,800 and the CRM has "Professional" at $2,400. You fix the CRM record. Total time from drift to resolution: five minutes. No quarterly audit. No spreadsheet. No week of detective work. The agent does not get bored. It does not skip a record because it is Friday afternoon. It checks every write, every time.

## Takeaway

The pattern is simple: compare on write, not on schedule. Any time two systems hold overlapping state, an agent that reconciles on every mutation will find drift in seconds instead of weeks. Apply this wherever you see a "sync" job that runs nightly, weekly, or quarterly. The cost of one API call per write is always cheaper than the cost of investigating stale drift after the fact.