## Opening thesis

You will build a closed-loop accountability system where every action your AI agent claims to perform gets recorded, verified, and timestamped through CueAPI. An agent can verify its own work at machine speed across hundreds of tasks simultaneously, which no human can do without a dashboard and hours of spot-checking. By the end, your agent will never say "done" without proof.

## Before

Your agent says it sent the email but you have no proof it actually did. You open the logs. The logs say `status: success`. You open your email provider's dashboard. You search for the recipient. You find nothing, or you find something from three hours ago and you are not sure it is the right one. You ask a teammate to check. They shrug. You write a Slack message asking the agent to re-send. The agent re-sends. Now the customer gets two emails. You spend forty minutes on a task that should have taken zero. This happens once a week at first. Then it happens daily as you add more automated workflows. You stop trusting the agent. You start spot-checking everything manually. The agent is fast, but your verification process is slow, so the bottleneck is you.

## Architecture

The system has four components. The agent performs actions (sending emails, creating records, calling APIs). After each action, it records the outcome as a "cue" in CueAPI, attaching the external receipt or ID from the downstream service. A verification loop queries CueAPI on a schedule to confirm that every dispatched action has a matching cue with a valid external reference. If a cue is missing or stale, the system flags it.

```text
DIAGRAM: Accountability loop architecture
Caption: Shows how the agent records proof of work and how the verifier checks it.
Nodes:
1. AI Agent - Performs actions (send email, create record, etc.)
2. Downstream Service - The external API that does the real work (email provider, CRM, etc.)
3. CueAPI - Stores timestamped cues with external_id references
4. Verifier Script - Queries CueAPI to confirm all actions have matching proof
5. Alert Channel - Receives notifications when verification fails (Slack, log, webhook)
Flow:
- AI Agent calls Downstream Service to perform an action
- Downstream Service returns a receipt or delivery ID
- AI Agent posts a cue to CueAPI with the receipt as external_id
- Verifier Script queries CueAPI for recent cues on a schedule
- Verifier Script flags missing or unverified cues to Alert Channel
```

## Step-by-step implementation

### Step 1: Set your CueAPI key

Sign up at https://www.cueapi.com and copy your API key from the dashboard. Store it as an environment variable. Every subsequent script reads from this variable.

```bash
export CUEAPI_API_KEY="cue_live_abc123your_real_key_here"
```

### Step 2: Define a cue schema for email actions

CueAPI organizes proof around "cue types." Create a cue type that represents an email-send action. This gives structure to every cue your agent will record. The `external_id_label` field tells anyone reading the cue what the external reference means.

```bash
curl -X POST https://api.cueapi.com/v1/cue-types \
  -H "Authorization: Bearer $CUEAPI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "email_sent",
    "description": "Proof that an email was dispatched and accepted by the mail provider",
    "external_id_label": "delivery_receipt_id",
    "retention_days": 90
  }'
```

### Step 3: Simulate the agent sending an email

In a real system, this is your agent calling SendGrid, Resend, Postmark, or any transactional email API. The key detail: the email provider returns a message ID or delivery receipt. Capture it. That receipt is your proof.

```python
import os
import uuid
import datetime

# Simulating the downstream email service response.
# In production, replace this with your actual email API call.
def send_email(to: str, subject: str, body: str) -> dict:
    """Calls your email provider. Returns the provider's receipt."""
    # Simulated response from the email provider
    return {
        "status": "accepted",
        "message_id": f"msg_{uuid.uuid4().hex[:16]}",
        "timestamp": datetime.datetime.utcnow().isoformat() + "Z"
    }

result = send_email(
    to="customer@example.org",
    subject="Your invoice is ready",
    body="Please find your invoice attached."
)

print(f"Email provider returned message_id: {result['message_id']}")
print(f"Status: {result['status']}")
```

### Step 4: Record the cue in CueAPI

Immediately after the email send succeeds, post a cue. The `external_id` field holds the email provider's message ID. This is the chain of evidence: your agent acted, the provider acknowledged, and CueAPI recorded the acknowledgment with a timestamp.

```python
import requests

CUEAPI_API_KEY = os.environ["CUEAPI_API_KEY"]

def record_cue(cue_type: str, external_id: str, metadata: dict) -> dict:
    response = requests.post(
        "https://api.cueapi.com/v1/cues",
        headers={
            "Authorization": f"Bearer {CUEAPI_API_KEY}",
            "Content-Type": "application/json"
        },
        json={
            "cue_type": cue_type,
            "external_id": external_id,
            "metadata": metadata,
            "status": "completed"
        }
    )
    response.raise_for_status()
    return response.json()

cue = record_cue(
    cue_type="email_sent",
    external_id=result["message_id"],
    metadata={
        "to": "customer@example.org",
        "subject": "Your invoice is ready",
        "provider_status": result["status"],
        "provider_timestamp": result["timestamp"]
    }
)

print(f"Cue recorded: {cue['id']} at {cue['created_at']}")
```

### Step 5: Build the verifier script

This script runs on a schedule (cron, Lambda, whatever you prefer). It pulls recent cues from CueAPI and checks that every expected action has a corresponding cue. If a cue is missing, it flags the gap.

```python
from datetime import datetime, timedelta

def get_recent_cues(cue_type: str, since_minutes: int = 60) -> list:
    since = (datetime.utcnow() - timedelta(minutes=since_minutes)).isoformat() + "Z"
    response = requests.get(
        "https://api.cueapi.com/v1/cues",
        headers={"Authorization": f"Bearer {CUEAPI_API_KEY}"},
        params={
            "cue_type": cue_type,
            "since": since,
            "status": "completed"
        }
    )
    response.raise_for_status()
    return response.json()["cues"]

recent_cues = get_recent_cues("email_sent", since_minutes=60)
print(f"Found {len(recent_cues)} verified email cues in the last hour.")

for c in recent_cues:
    print(f"  Cue {c['id']}: external_id={c['external_id']}, created={c['created_at']}")
```

### Step 6: Cross-reference expected actions against recorded cues

Your agent tracks which actions it dispatched. The verifier compares that list to the cues in CueAPI. Any action without a matching cue is a gap. This is the core of the accountability loop.

```python
# In production, this list comes from your agent's task queue or database.
expected_actions = [
    {"type": "email_sent", "external_id": result["message_id"], "description": "Invoice email to customer@example.org"},
    {"type": "email_sent", "external_id": "msg_does_not_exist", "description": "Welcome email to newuser@example.org"}
]

recorded_external_ids = {c["external_id"] for c in recent_cues}

gaps = []
for action in expected_actions:
    if action["external_id"] not in recorded_external_ids:
        gaps.append(action)

if gaps:
    print(f"ALERT: {len(gaps)} action(s) have no proof in CueAPI:")
    for g in gaps:
        print(f"  MISSING: {g['description']} (expected external_id: {g['external_id']})")
else:
    print("All actions verified. No gaps.")
```

### Step 7: Send alerts for gaps

When the verifier finds a gap, push a notification. This example posts to a Slack webhook. Replace the URL with your own. The point: no gap goes unnoticed.

```python
import json

SLACK_WEBHOOK_URL = os.environ.get("SLACK_WEBHOOK_URL", "")

def alert_gaps(gaps: list):
    if not gaps or not SLACK_WEBHOOK_URL:
        return
    message = f":warning: Agent accountability gap detected. {len(gaps)} action(s) missing proof:\n"
    for g in gaps:
        message += f"  - {g['description']} (external_id: {g['external_id']})\n"
    requests.post(SLACK_WEBHOOK_URL, json={"text": message})
    print("Alert sent to Slack.")

alert_gaps(gaps)
```

### Step 8: Schedule the verifier

Run the verifier every 15 minutes using cron. Save Steps 5 through 7 in a single file called `verify_cues.py`. The cron entry below assumes the script and environment variables are available at the specified path.

```bash
# Add to crontab with: crontab -e
*/15 * * * * cd /opt/agent-accountability && /usr/bin/python3 verify_cues.py >> /var/log/cue_verifier.log 2>&1
```

## Breakage

Skip the cue-recording step. Your agent sends the email, gets back a message ID, and discards it. The downstream service accepted the request, but you have no record on your side. A week later, a customer complains they never received the invoice. You search your agent logs. The logs say "email sent." You search the email provider dashboard. The provider purged delivery records after 72 hours. You have no proof. You cannot tell if the email was sent, bounced, or never dispatched. You re-send manually and apologize. The trust gap widens.

```text
DIAGRAM: Breakage point without accountability
Caption: Shows where proof is lost when the agent does not record a cue.
Nodes:
1. AI Agent - Sends email, discards receipt
2. Downstream Service - Accepts request, returns message_id
3. Agent Logs - Contains only "status: success" with no external reference
4. Human Operator - Has no way to verify the action happened
Flow:
- AI Agent calls Downstream Service
- Downstream Service returns message_id to AI Agent
- AI Agent logs "success" but does not store message_id durably
- Human Operator checks logs, finds no external proof
- Human Operator checks Downstream Service dashboard, finds records expired
- Verification fails completely
```

## The fix

The fix is the cue-recording call from Step 4. Insert it directly after the email send, inside the same try block. If the cue recording fails, the agent retries. If the retry fails, the agent logs a warning so the verifier catches it as a gap. Here is the complete agent action function combining Steps 3 and 4 into a single atomic operation.

```python
import time

def send_email_with_proof(to: str, subject: str, body: str, max_retries: int = 3) -> dict:
    # Step 1: Send the email
    result = send_email(to, subject, body)

    # Step 2: Record the cue, with retries
    cue = None
    for attempt in range(max_retries):
        try:
            cue = record_cue(
                cue_type="email_sent",
                external_id=result["message_id"],
                metadata={
                    "to": to,
                    "subject": subject,
                    "provider_status": result["status"],
                    "provider_timestamp": result["timestamp"]
                }
            )
            break
        except requests.exceptions.RequestException as e:
            print(f"Cue recording attempt {attempt + 1} failed: {e}")
            time.sleep(2 ** attempt)

    if cue is None:
        print(f"WARNING: Email sent (message_id={result['message_id']}) but cue recording failed after {max_retries} attempts.")

    return {"email_result": result, "cue": cue}

proof = send_email_with_proof(
    to="customer@example.org",
    subject="Your invoice is ready",
    body="Please find your invoice attached."
)
print(f"Action complete. Cue ID: {proof['cue']['id'] if proof['cue'] else 'MISSING'}")
```

## Fixed state

```text
DIAGRAM: Accountability loop with fix in place
Caption: Shows the complete loop where every action produces durable, verifiable proof.
Nodes:
1. AI Agent - Sends email, then immediately records cue with retries
2. Downstream Service - Returns delivery receipt (message_id)
3. CueAPI - Stores timestamped cue with message_id as external_id
4. Verifier Script - Runs every 15 minutes, cross-references expected actions against cues
5. Alert Channel - Receives notification only when a gap is detected
6. Human Operator - Reviews alerts only, not every action
Flow:
- AI Agent calls Downstream Service to send email
- Downstream Service returns message_id
- AI Agent posts cue to CueAPI with message_id as external_id
- CueAPI confirms cue stored with timestamp
- Verifier Script queries CueAPI for cues since last check
- Verifier Script compares cues against expected actions from agent task queue
- If gap found, Verifier Script posts alert to Alert Channel
- Human Operator checks Alert Channel, not individual actions
```

## After

CueAPI has timestamped evidence the email was delivered, with the delivery receipt as the external_id. You do not open the email provider dashboard. You do not ask a teammate to check. You do not re-send anything manually. The verifier runs every 15 minutes and checks hundreds of actions in under two seconds. When a gap appears, you get a Slack notification with the exact action that failed and the external ID that should have been recorded. You fix the one broken thing instead of spot-checking everything. The agent is fast, and now your verification is fast too.

## Takeaway

The pattern is simple: every agent action that touches an external system must produce a durable receipt, stored outside the agent's own logs, immediately after the action completes. The receipt includes the external system's own identifier so you can trace the chain end to end. Apply this to any action, not just emails. If your agent creates a support ticket, store the ticket ID as a cue. If it posts to a webhook, store the response ID. Proof of work is not optional when the worker is autonomous.
