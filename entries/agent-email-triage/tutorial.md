## Opening thesis

You will build an agent that reads your Gmail inbox, classifies every message by urgency and topic, drafts replies for the important ones, and queues everything for your approval before a single byte leaves your outbox. A human triages 200 emails in 90 minutes with declining accuracy as fatigue sets in; an agent triages 200 emails in 30 seconds with consistent classification and gates every send through human review. The agent does not get tired at email 150. You do.

## Before

Your inbox has 247 unread emails. You spend an hour every morning sorting them and still miss the important ones because they got buried under newsletters, sales pitches, and automated alerts from services you forgot you signed up for. By the time you reach email 80, your eyes glaze. You start skimming subject lines instead of reading bodies. A client follow-up slips past because the subject line said "Quick question" and your brain filed it next to the seventeen other quick questions that turned out to be nothing. You find it three days later. The client found someone else.

## Architecture

The system has five components. Gmail provides the raw messages. A Python scheduler polls for new mail every 15 minutes. Claude classifies each message and drafts replies for high-priority items. A local SQLite database stores classifications and drafts. A review script shows you pending drafts and sends only what you approve.

```text
DIAGRAM: Email Triage Agent
Caption: Data flow from Gmail inbox through classification to human-approved sending
Nodes:
1. Gmail API - fetches unread messages, sends approved replies
2. Scheduler (cron) - triggers fetch every 15 minutes
3. Classifier (Claude API) - reads each email, assigns category and priority, drafts replies
4. SQLite DB - stores message metadata, classifications, draft replies, approval status
5. Review CLI - displays pending drafts, accepts approve/reject/edit from human
Flow:
- Scheduler triggers Gmail API fetch of unread messages
- Each message passes to Classifier with sender, subject, and body
- Classifier returns category, priority (1-5), and draft reply (if priority >= 4)
- All results write to SQLite DB with status "pending"
- Review CLI reads pending items from SQLite DB
- Human approves, edits, or rejects each draft
- Approved drafts send via Gmail API
- SQLite DB updates status to "sent" or "rejected"
```

## Step-by-step implementation

### Step 1: Set up Gmail API credentials

You need OAuth 2.0 credentials for the Gmail API. Go to https://console.cloud.google.com/apis/credentials, create a project, enable the Gmail API, and download the OAuth client JSON file. Save it as `credentials.json` in your project directory. The first run will open a browser window for you to authorize the app.

```bash
mkdir email-triage && cd email-triage
pip install google-auth google-auth-oauthlib google-auth-httplib2 google-api-python-client anthropic
export ANTHROPIC_API_KEY="sk-ant-..."  # Get yours at https://console.anthropic.com/settings/keys
```

### Step 2: Create the database schema

The SQLite database holds every classified email and its draft reply. The `status` column controls what appears in your review queue. Nothing sends without an explicit status change to `approved`.

```python
# db_setup.py
import sqlite3

def init_db(path="triage.db"):
    conn = sqlite3.connect(path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS emails (
            id TEXT PRIMARY KEY,
            sender TEXT,
            subject TEXT,
            body TEXT,
            category TEXT,
            priority INTEGER,
            draft_reply TEXT,
            status TEXT DEFAULT 'pending',
            classified_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    conn.close()

if __name__ == "__main__":
    init_db()
    print("Database initialized.")
```

### Step 3: Build the Gmail fetch module

This module authenticates with Gmail and pulls unread messages. It marks each fetched message as read so the next poll skips it. The function returns a list of dicts with `id`, `sender`, `subject`, and `body`.

```python
# gmail_fetch.py
import os
import base64
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build

SCOPES = [
    "https://www.googleapis.com/auth/gmail.readonly",
    "https://www.googleapis.com/auth/gmail.modify",
    "https://www.googleapis.com/auth/gmail.send",
]

def get_service():
    creds = None
    if os.path.exists("token.json"):
        creds = Credentials.from_authorized_user_file("token.json", SCOPES)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file("credentials.json", SCOPES)
            creds = flow.run_local_server(port=0)
        with open("token.json", "w") as token:
            token.write(creds.to_json())
    return build("gmail", "v1", credentials=creds)

def fetch_unread(max_results=50):
    service = get_service()
    results = service.users().messages().list(
        userId="me", q="is:unread", maxResults=max_results
    ).execute()
    messages = results.get("messages", [])
    emails = []
    for msg in messages:
        full = service.users().messages().get(userId="me", id=msg["id"], format="full").execute()
        headers = {h["name"]: h["value"] for h in full["payload"]["headers"]}
        body = ""
        if "parts" in full["payload"]:
            for part in full["payload"]["parts"]:
                if part["mimeType"] == "text/plain" and "data" in part.get("body", {}):
                    body = base64.urlsafe_b64decode(part["body"]["data"]).decode("utf-8")
                    break
        elif "data" in full["payload"].get("body", {}):
            body = base64.urlsafe_b64decode(full["payload"]["body"]["data"]).decode("utf-8")
        emails.append({
            "id": msg["id"],
            "sender": headers.get("From", ""),
            "subject": headers.get("Subject", ""),
            "body": body[:3000],
        })
        service.users().messages().modify(
            userId="me", id=msg["id"], body={"removeLabelIds": ["UNREAD"]}
        ).execute()
    return emails
```

### Step 4: Build the classifier

This is the core of the agent. Each email goes to Claude with a structured prompt. The model returns JSON with a `category`, a `priority` score from 1 to 5, and a `draft_reply` for anything priority 4 or above. The prompt enforces the output schema so you can parse it reliably.

```python
# classifier.py
import os
import json
import anthropic

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

SYSTEM_PROMPT = """You are an email triage assistant. For each email, return a JSON object with exactly three fields:
- "category": one of "client", "internal", "billing", "newsletter", "sales", "notification", "personal", "spam"
- "priority": integer from 1 (ignore) to 5 (urgent, needs reply today)
- "draft_reply": a short, professional reply if priority >= 4, otherwise null

Return only valid JSON. No markdown fences. No commentary."""

def classify(email):
    user_msg = f"From: {email['sender']}\nSubject: {email['subject']}\n\n{email['body']}"
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": user_msg}],
    )
    text = response.content[0].text.strip()
    try:
        result = json.loads(text)
    except json.JSONDecodeError:
        result = {"category": "unknown", "priority": 1, "draft_reply": None}
    return result
```

### Step 5: Wire fetch and classify into a single run

This script is what cron calls. It fetches unread mail, classifies each message, and writes results to SQLite. One run, one pass, no state between runs except the database.

```python
# triage_run.py
import sqlite3
from gmail_fetch import fetch_unread
from classifier import classify
from db_setup import init_db

def run():
    init_db()
    conn = sqlite3.connect("triage.db")
    emails = fetch_unread(max_results=50)
    for email in emails:
        existing = conn.execute("SELECT id FROM emails WHERE id = ?", (email["id"],)).fetchone()
        if existing:
            continue
        result = classify(email)
        conn.execute(
            "INSERT INTO emails (id, sender, subject, body, category, priority, draft_reply, status) VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
            (email["id"], email["sender"], email["subject"], email["body"],
             result["category"], result["priority"], result.get("draft_reply"), "pending")
        )
        conn.commit()
        print(f"Classified: [{result['category']}] P{result['priority']} - {email['subject']}")
    conn.close()
    print(f"Processed {len(emails)} emails.")

if __name__ == "__main__":
    run()
```

### Step 6: Schedule the agent with cron

This cron entry runs the triage script every 15 minutes. Adjust the paths to match your environment. The agent accumulates results in the database silently. You review when you choose.

```bash
crontab -e
# Add this line (adjust paths):
*/15 * * * * cd /home/you/email-triage && /home/you/.venv/bin/python triage_run.py >> triage.log 2>&1
```

### Step 7: Build the review CLI

This is the gate. The review script shows you every pending draft reply. You type `y` to approve, `n` to reject, or `e` to edit. Nothing sends without your explicit input. This is the entire point of the system.

```python
# review.py
import sqlite3
import base64
from email.mime.text import MIMEText
from gmail_fetch import get_service

def send_reply(service, to, subject, body, thread_id):
    message = MIMEText(body)
    message["to"] = to
    message["subject"] = f"Re: {subject}"
    raw = base64.urlsafe_b64encode(message.as_bytes()).decode()
    service.users().messages().send(
        userId="me", body={"raw": raw, "threadId": thread_id}
    ).execute()

def review():
    conn = sqlite3.connect("triage.db")
    conn.row_factory = sqlite3.Row
    pending = conn.execute(
        "SELECT * FROM emails WHERE draft_reply IS NOT NULL AND status = 'pending' ORDER BY priority DESC"
    ).fetchall()
    if not pending:
        print("No pending drafts to review.")
        return
    service = get_service()
    print(f"\n{len(pending)} drafts waiting for review.\n")
    for row in pending:
        print(f"--- Priority {row['priority']} | {row['category']} ---")
        print(f"From: {row['sender']}")
        print(f"Subject: {row['subject']}")
        print(f"\nDraft reply:\n{row['draft_reply']}\n")
        choice = input("[y]approve  [n]reject  [e]edit > ").strip().lower()
        if choice == "y":
            send_reply(service, row["sender"], row["subject"], row["draft_reply"], row["id"])
            conn.execute("UPDATE emails SET status = 'sent' WHERE id = ?", (row["id"],))
            print("Sent.")
        elif choice == "e":
            print("Enter new reply (end with a blank line):")
            lines = []
            while True:
                line = input()
                if line == "":
                    break
                lines.append(line)
            edited = "\n".join(lines)
            send_reply(service, row["sender"], row["subject"], edited, row["id"])
            conn.execute("UPDATE emails SET draft_reply = ?, status = 'sent' WHERE id = ?", (edited, row["id"]))
            print("Sent edited reply.")
        else:
            conn.execute("UPDATE emails SET status = 'rejected' WHERE id = ?", (row["id"],))
            print("Rejected.")
        conn.commit()
    conn.close()

if __name__ == "__main__":
    review()
```

### Step 8: Add a daily summary report

Once a day you want to see what the agent processed. This script prints a breakdown by category and priority. It tells you where your inbox attention actually goes, which is information you never had when you were doing it by hand.

```python
# summary.py
import sqlite3
from datetime import datetime, timedelta

def daily_summary():
    conn = sqlite3.connect("triage.db")
    yesterday = (datetime.now() - timedelta(days=1)).isoformat()
    rows = conn.execute(
        "SELECT category, priority, status, COUNT(*) as cnt FROM emails WHERE classified_at >= ? GROUP BY category, priority, status ORDER BY priority DESC",
        (yesterday,)
    ).fetchall()
    print(f"\nTriage summary since {yesterday[:10]}\n")
    print(f"{'Category':<15} {'Priority':<10} {'Status':<10} {'Count':<5}")
    print("-" * 42)
    for cat, pri, status, cnt in rows:
        print(f"{cat:<15} {pri:<10} {status:<10} {cnt:<5}")
    total = conn.execute(
        "SELECT COUNT(*) FROM emails WHERE classified_at >= ?", (yesterday,)
    ).fetchone()[0]
    print(f"\nTotal: {total} emails classified.")
    conn.close()

if __name__ == "__main__":
    daily_summary()
```

## Breakage

If you skip the review step and wire the classifier directly to Gmail's send endpoint, you will eventually send something embarrassing. The classifier might draft a terse "Noted, thanks" to your CEO's urgent request. It might reply to a newsletter with a confused acknowledgment. It might interpret sarcasm literally and send a sincere apology to a joke email from a coworker. The model has no understanding of your relationships, your org's politics, or the tone appropriate to each sender. Without the human gate, the system becomes a reputation risk.

```text
DIAGRAM: Breakage, no human review
Caption: Classifier sends replies directly, bypassing human judgment
Nodes:
1. Gmail API - fetches unread messages, sends replies
2. Classifier (Claude API) - classifies and drafts replies
3. SQLite DB - stores results but status is ignored
Flow:
- Gmail API fetches unread messages
- Classifier generates draft replies
- Classifier output goes directly to Gmail API send endpoint
- FAILURE POINT: no human reviews the draft before sending
- Embarrassing reply reaches recipient
```

## The fix

The fix is already built into Step 7. The `review.py` script enforces a hard gate: every draft sits in the database with status `pending` until a human explicitly changes it. The key constraint is in the database schema and the send logic. No code path exists that moves a message from `pending` to `sent` without user input. But we can make this even stronger by adding a safety check inside the send function itself.

```python
# Add this validation to the top of send_reply in review.py
def send_reply_safe(service, to, subject, body, thread_id, email_id, conn):
    status = conn.execute("SELECT status FROM emails WHERE id = ?", (email_id,)).fetchone()
    if status is None or status[0] != "approved":
        raise ValueError(f"Cannot send email {email_id}: status is {status}, not approved.")
    message = MIMEText(body)
    message["to"] = to
    message["subject"] = f"Re: {subject}"
    raw = base64.urlsafe_b64encode(message.as_bytes()).decode()
    service.users().messages().send(
        userId="me", body={"raw": raw, "threadId": thread_id}
    ).execute()
    conn.execute("UPDATE emails SET status = 'sent' WHERE id = ?", (email_id,))
    conn.commit()
```

Now the review flow becomes: user types `y`, the script sets status to `approved`, then calls `send_reply_safe`, which double-checks the status before touching the Gmail API. Two locks on the same door. The paranoid version is the correct version.

## Fixed state

```text
DIAGRAM: Full system with human gate
Caption: Every draft passes through human review before sending
Nodes:
1. Gmail API - fetches unread messages, sends only approved replies
2. Scheduler (cron) - triggers fetch every 15 minutes
3. Classifier (Claude API) - classifies and drafts replies
4. SQLite DB - stores all data, enforces pending/approved/sent status
5. Review CLI - human approves, edits, or rejects each draft
6. send_reply_safe - validates approved status before sending
Flow:
- Scheduler triggers Gmail API fetch
- Messages pass to Classifier
- Results write to SQLite DB with status "pending"
- Review CLI displays pending drafts to human
- Human sets status to "approved" or "rejected"
- send_reply_safe checks status equals "approved" before calling Gmail API
- Gmail API sends only verified, human-approved replies
- SQLite DB updates status to "sent"
```

## After

Your inbox still gets 247 emails. The agent reads them every 15 minutes, classifies each by importance and topic, and drafts responses to the high-priority ones. At 9 AM you open the review CLI. Twelve drafts are waiting, sorted by priority. You approve nine, edit two, reject one. Total time: four minutes. The newsletters are tagged and archived. The sales pitches are tagged and ignored. The client follow-up that says "Quick question" is priority 5, sitting at the top of your queue, with a draft reply that you adjust and send. You did not miss it. You did not have to read 200 other emails to find it.

## Takeaway

The pattern is classification plus human gate. Let the machine do the work that degrades with volume and fatigue: reading, sorting, drafting. Keep the human at the chokepoint where judgment matters: the send button. This pattern applies to any communication system where the cost of a bad output is higher than the cost of a five-second review.