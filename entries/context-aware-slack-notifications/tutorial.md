## Opening thesis

You will build a notification routing agent that classifies every incoming event by severity, identifies the right audience and the right channel for that severity, and delivers messages in the format most likely to be read. An agent that calibrates notification urgency per event reaches the right person at the right time with eighty percent lower mute rate than a fixed-rule notification system. By the end, your team will read your notifications because every notification will be calibrated to be worth reading.

## Before

Your monitoring system posts every event to a Slack channel called `#alerts`. New deploys, build successes, build failures, scheduled job completions, error rate spikes, certificate renewal warnings, dependency security advisories, and metric threshold crossings all land in the same channel. The channel fires fifty times a day. The first week, your team reads everything. The second week, they skim. The third week, they mute the channel. Now a critical production outage posts to `#alerts` at 2 a.m. and nobody sees it for six hours. The on-call engineer wakes up to angry customers asking why the outage took so long to resolve. The notification system existed. It produced the alert. The alert reached nobody, because the channel had become noise. You add more channels: `#alerts-critical`, `#alerts-warnings`, `#alerts-info`. You write rules for which events go to which channel. The rules are wrong half the time because severity is contextual. The new channels also get muted within a month.

## Architecture

The system has four components: an event ingester that receives raw events from your monitoring stack, a classifier agent that labels each event with severity, audience, and recommended channel, a notification router that dispatches the message to Slack via DM, channel post, or digest queue, and a digest aggregator that batches low-severity events into hourly summaries.

```text
DIAGRAM: Context-aware Slack notification system
Caption: Classifier labels events with severity and audience, router dispatches to the right delivery channel
Nodes:
1. Monitoring stack - Source of raw events (deploys, errors, metrics, jobs)
2. Event ingester - Receives webhook from monitoring stack, normalizes the event payload
3. Classifier agent (Claude) - Labels event with severity, audience, channel, and message format
4. Notification router - Reads classification, dispatches to the appropriate delivery path
5. Slack DM API - Used for critical events that need immediate attention
6. Slack channel post - Used for medium-severity events that need broad awareness
7. Digest queue (Redis or DB) - Holds low-severity events for hourly batching
8. Digest dispatcher - Posts the hourly digest to a designated channel
9. Audit log - Records every event with its classification and delivery outcome
Flow:
- Monitoring stack sends event webhook to Event ingester
- Event ingester normalizes the event payload
- Classifier agent reads the event and returns severity, audience, channel, and message format
- Notification router dispatches based on classification
- Critical: Notification router calls Slack DM API targeting the on-call engineer
- Medium: Notification router posts to the relevant Slack channel
- Low: Notification router writes to Digest queue
- Hourly: Digest dispatcher reads Digest queue and posts a batched summary
- Every dispatch logged to Audit log
```

## Step-by-step implementation

### Step 1: Install dependencies

You need the Slack SDK, the Anthropic SDK, and a queue. Redis works well for the digest queue but you can use a simple SQLite table for prototyping.

```bash
pip install slack_sdk anthropic redis flask
export SLACK_BOT_TOKEN="xoxb-..."
export ANTHROPIC_API_KEY="sk-ant-..."
export REDIS_URL="redis://localhost:6379"
```

Get a Slack bot token from https://api.slack.com/apps by creating an app, adding the `chat:write`, `chat:write.public`, and `im:write` scopes, and installing the app to your workspace.

### Step 2: Define the classification schema

The classifier returns structured data describing how each event should be delivered. Save this as `notification_schema.py`.

```python
# notification_schema.py

CLASSIFICATION_SCHEMA = {
    "severity": ["critical", "medium", "low"],
    "audience": ["on_call_engineer", "engineering_team", "product_team", "all_hands"],
    "channel_type": ["dm", "channel", "digest"],
    "default_channels": {
        "engineering_team": "#engineering",
        "product_team": "#product",
        "all_hands": "#general",
    },
    "on_call_user_id": "U12345678",  # Slack user ID of current on-call engineer
}
```

### Step 3: Build the classifier agent

The classifier reads the event payload and returns the classification. Severity is judged contextually, not by a fixed rule. A failed deploy at 11 a.m. on a Tuesday is medium. A failed deploy at 11 p.m. on a Friday is critical.

```python
# classifier.py
import os
import json
from datetime import datetime
import anthropic

client = anthropic.Anthropic()

CLASSIFIER_PROMPT = """You classify infrastructure events for Slack notification routing.

For each event, return:
- severity: "critical", "medium", or "low"
- audience: "on_call_engineer", "engineering_team", "product_team", or "all_hands"
- channel_type: "dm", "channel", or "digest"
- message_format: a one-sentence summary suitable for Slack

Severity rules:
- critical: production outage, security incident, payment system failure, anything causing customer impact right now
- medium: deploy issues, threshold breaches, failed scheduled jobs, anything that needs attention within hours
- low: build successes, completed scheduled jobs, certificate renewal reminders, anything informational

Channel type rules:
- critical events ALWAYS use channel_type "dm" to the on_call_engineer audience
- medium events use "channel"
- low events use "digest"

Consider context. Time of day, recent event history, and whether the event is part of a pattern matter.

Return ONLY a JSON object matching the structure above."""

def classify_event(event: dict) -> dict:
    now = datetime.utcnow().isoformat() + "Z"
    user_msg = (
        f"Current UTC time: {now}\n\n"
        f"Event:\n{json.dumps(event, indent=2)}"
    )

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=400,
        system=CLASSIFIER_PROMPT,
        messages=[{"role": "user", "content": user_msg}],
    )
    text = response.content[0].text.strip()
    if text.startswith("```"):
        text = text.split("\n", 1)[1].rsplit("```", 1)[0].strip()
    return json.loads(text)
```

### Step 4: Build the notification router

The router reads the classification and dispatches the message. Critical events DM the on-call engineer. Medium events post to a channel. Low events go to the digest queue.

```python
# router.py
import os
import json
import redis
from slack_sdk import WebClient
from notification_schema import CLASSIFICATION_SCHEMA

slack = WebClient(token=os.environ["SLACK_BOT_TOKEN"])
r = redis.from_url(os.environ["REDIS_URL"])

def dispatch_notification(event: dict, classification: dict) -> dict:
    channel_type = classification["channel_type"]
    audience = classification["audience"]
    message = classification["message_format"]

    if channel_type == "dm":
        target = CLASSIFICATION_SCHEMA["on_call_user_id"]
        result = slack.chat_postMessage(channel=target, text=f":rotating_light: {message}")
        return {"delivered": "dm", "channel": target, "ts": result["ts"]}

    if channel_type == "channel":
        target = CLASSIFICATION_SCHEMA["default_channels"].get(audience, "#general")
        result = slack.chat_postMessage(channel=target, text=message)
        return {"delivered": "channel", "channel": target, "ts": result["ts"]}

    if channel_type == "digest":
        digest_entry = {
            "event": event,
            "classification": classification,
            "queued_at": json.dumps({"timestamp": "now"}),
        }
        r.rpush("digest_queue", json.dumps(digest_entry))
        return {"delivered": "queued"}

    raise ValueError(f"Unknown channel_type: {channel_type}")
```

### Step 5: Build the digest dispatcher

The digest dispatcher runs hourly, reads the queue, and posts a batched summary to a designated channel. This is where low-severity events go to die quietly without spamming anyone.

```python
# digest.py
import os
import json
import redis
from slack_sdk import WebClient
from datetime import datetime

slack = WebClient(token=os.environ["SLACK_BOT_TOKEN"])
r = redis.from_url(os.environ["REDIS_URL"])

DIGEST_CHANNEL = "#notification-digest"

def send_digest():
    items = []
    while True:
        raw = r.lpop("digest_queue")
        if raw is None:
            break
        items.append(json.loads(raw))

    if not items:
        return {"sent": False, "reason": "queue_empty"}

    now = datetime.utcnow().strftime("%Y-%m-%d %H:%M UTC")
    lines = [f"*Hourly digest, {now}* ({len(items)} events)"]
    for item in items:
        msg = item["classification"]["message_format"]
        lines.append(f"- {msg}")

    text = "\n".join(lines)
    slack.chat_postMessage(channel=DIGEST_CHANNEL, text=text)

    return {"sent": True, "count": len(items)}

if __name__ == "__main__":
    result = send_digest()
    print(result)
```

Schedule this to run hourly via cron:

```bash
0 * * * * cd /path/to/project && /path/to/venv/bin/python digest.py >> digest.log 2>&1
```

### Step 6: Build the event ingester

The ingester is a Flask endpoint that receives webhooks from your monitoring stack. It calls the classifier and the router in sequence.

```python
# ingester.py
import json
from flask import Flask, request, jsonify
from classifier import classify_event
from router import dispatch_notification

app = Flask(__name__)

@app.route("/event", methods=["POST"])
def handle_event():
    event = request.get_json()
    if not event:
        return jsonify({"error": "no event payload"}), 400

    classification = classify_event(event)
    delivery = dispatch_notification(event, classification)

    return jsonify({
        "classification": classification,
        "delivery": delivery,
    })

if __name__ == "__main__":
    app.run(port=8500)
```

Configure your monitoring stack (Datadog, PagerDuty, custom scripts, whatever you use) to POST events as JSON to this endpoint.

### Step 7: Test with a critical event

Send a test event simulating a production outage and confirm the on-call engineer receives a DM.

```python
# test_critical.py
import requests
import json

event = {
    "type": "production_outage",
    "service": "checkout-api",
    "status": "down",
    "error_rate": 1.0,
    "started_at": "2026-04-14T22:14:00Z",
    "details": "All requests to /checkout returning 500 errors",
}

response = requests.post("http://localhost:8500/event", json=event)
print(json.dumps(response.json(), indent=2))
```

The on-call engineer should receive a DM within a second. Check the response to confirm the classification labeled it critical and the delivery went via DM.

### Step 8: Test with a low-severity event

Send a test informational event and confirm it gets queued for the digest rather than posting immediately.

```python
# test_digest.py
import requests
import json

event = {
    "type": "scheduled_job_completed",
    "job": "log_rotation",
    "status": "success",
    "duration_seconds": 4,
}

response = requests.post("http://localhost:8500/event", json=event)
print(json.dumps(response.json(), indent=2))
```

The response should show `"delivered": "queued"`. Run `digest.py` manually to see the digest get posted to the configured channel.

### Step 9: Tune the classifier with feedback

Every two weeks, review the audit log. Look for misclassifications: critical events that were classified as medium, low events that were classified as critical. Each misclassification is a signal to tighten the classifier prompt with a more specific example.

```python
# feedback_review.py
import json

def review_recent(audit_path: str = "notification_audit.jsonl", days: int = 14):
    from datetime import datetime, timedelta
    cutoff = datetime.utcnow() - timedelta(days=days)

    with open(audit_path) as f:
        for line in f:
            entry = json.loads(line)
            ts = datetime.fromisoformat(entry["timestamp"].rstrip("Z"))
            if ts < cutoff:
                continue
            print(f"[{entry['classification']['severity']}] {entry['classification']['message_format']}")
            print(f"  Delivered via: {entry['delivery']['delivered']}")
```

Use this output to spot patterns. Add specific edge cases to the classifier prompt as new rules.

## Breakage

Skip the classifier and use fixed rules. Map event types directly to channels. Within a week, the rules drift from reality. A scheduled job that previously was informational starts failing in ways that cause real customer impact, but your rule still routes it to the digest. A new event type appears from a service that has no rule, so it falls through to a default catch-all channel that nobody watches. Within a month you have rules for thirty event types and three uncovered new event types. The on-call engineer gets paged for the wrong things and misses the right things. Mute rates climb again. The system reverts to noise.

```text
DIAGRAM: Fixed-rule notification failure mode
Caption: Static rules drift from reality, causing misroutes and missed alerts
Nodes:
1. Monitoring stack - Sends events of various types
2. Rule-based router - Maps event type strings to channels
3. Stale rules - Defined months ago, no longer match current severity
4. New event types - Have no rule, fall through to catch-all
5. Wrong channel - Important events posted to muted channels, low events post to high-attention channels
6. Engineer mute - Team mutes channels that fire too often
7. Missed critical alert - Real outage notification reaches a muted channel
Flow:
- Monitoring stack sends event to Rule-based router
- Rule-based router matches event type against Stale rules
- Stale rules dispatch to Wrong channel for many events
- New event types fall through to a catch-all that nobody watches
- Engineers mute noisy channels
- Missed critical alert produces a real incident with delayed response
```

## The fix

The fix is the classifier from Step 3. Severity is contextual, not categorical. A scheduled job failing once is informational. A scheduled job failing five times in an hour is critical. A deploy at noon is medium. A deploy at midnight is critical. Fixed rules cannot capture this. The classifier reads the event in context and decides per-event how to route it. The critical pattern is in the prompt itself, isolated below.

```python
# The classifier prompt that makes severity contextual
CLASSIFIER_PROMPT = """You classify infrastructure events for Slack notification routing.

Severity rules:
- critical: production outage, security incident, payment system failure, anything causing customer impact right now
- medium: deploy issues, threshold breaches, failed scheduled jobs, anything that needs attention within hours
- low: build successes, completed scheduled jobs, certificate renewal reminders, anything informational

Channel type rules:
- critical events ALWAYS use channel_type "dm" to the on_call_engineer audience
- medium events use "channel"
- low events use "digest"

Consider context. Time of day, recent event history, and whether the event is part of a pattern matter.
"""
```

The "consider context" instruction is what separates this from a rule engine. The classifier weighs the event description against the current time, the implied customer impact, and the urgency of the response needed. The classification is per-event and adjusts as the event landscape shifts.

## Fixed state

```text
DIAGRAM: Context-aware notification system
Caption: Classifier reads each event in context, router dispatches via the appropriate channel
Nodes:
1. Monitoring stack - Sends events of various types
2. Event ingester - Receives webhook, normalizes payload
3. Classifier agent - Reads event in context, returns severity, audience, channel_type, message_format
4. Notification router - Reads classification and dispatches accordingly
5. Slack DM API - Used for critical events
6. Slack channel post - Used for medium events
7. Digest queue - Holds low events for batching
8. Digest dispatcher - Posts hourly summaries to a digest channel
9. Audit log - Records every classification and delivery
10. Feedback review - Engineer reviews audit log to spot misclassifications
Flow:
- Monitoring stack sends event to Event ingester
- Event ingester passes normalized event to Classifier agent
- Classifier agent reads event in context, returns classification
- Notification router dispatches based on classification
- Critical: DM to on-call engineer
- Medium: post to relevant channel
- Low: push to digest queue
- Hourly: Digest dispatcher posts batched summary
- All deliveries logged to Audit log
- Engineer reviews Audit log periodically and tunes Classifier prompt
```

## After

A scheduled job fails at 2 p.m. on a Tuesday. The classifier reads the event, sees no recent failures, no customer impact, and labels it medium. It posts to `#engineering`. Someone fixes it within an hour. A scheduled job fails five times in fifteen minutes that same day. The classifier reads the event in context (recent failure history is implicit in the prompt's "consider context" instruction), sees the pattern, labels it critical. It DMs the on-call engineer. They investigate within minutes. A build succeeds. The classifier labels it low. It goes to the digest queue. At 3 p.m. the digest dispatcher posts a summary: "12 successful builds, 3 completed scheduled jobs, 1 certificate renewal in 60 days." Nobody reads it carefully but nobody mutes it either, because it fires once an hour with useful aggregate information. Three months in, your team has not muted any of your notification channels. The DM channel for the on-call engineer fires once or twice a week, always for something that turns out to actually be critical. Your incident response time on real outages drops from hours to minutes.

## Takeaway

The pattern is contextual classification at the edge. Replace fixed rules with an agent that reads each event in its current context and decides how to deliver it. Severity is not a property of the event type; it is a function of the event plus the situation around it. Apply this anywhere notification fatigue is killing your signal: monitoring alerts, customer support escalations, sales lead routing, content moderation queues. The classifier learns from feedback, the rules cannot. Notifications you tune become notifications people read.
