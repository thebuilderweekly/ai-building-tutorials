## Opening thesis

You will build an agent that verifies every scheduled cron job actually ran, compares each outcome to an expected state, and alerts you within seconds when something fails. An agent can check a hundred cron outcomes per second against expected state, which a human doing daily ops review does in thirty minutes and misses silently. The system uses CueAPI to schedule verification checks that fire after each cron job's expected completion window.

## Before

You have ten cron jobs. They run daily. They do things like sync inventory, generate invoices, rotate logs, and push reports to an SFTP server. You trust them because they ran yesterday. One Tuesday, the invoice job fails. The error goes to a log file nobody reads. The cron daemon reports exit code 0 because your wrapper script swallows errors. Five days pass. A customer emails asking where their invoice is. You check the logs. The job has been failing since Tuesday. You fix it, re-run it manually, and promise yourself you will build monitoring. You do not build monitoring. This cycle repeats every quarter with a different job.

## Architecture

The system has three layers. First, your existing cron jobs, which write outcomes to a shared state store (a database table or a JSON file on disk). Second, a CueAPI scheduled task for each cron job, offset by the job's expected duration plus a buffer. Third, a verification agent that reads the outcome from the state store, compares it to the expected state, and fires an alert if something is wrong.

```text
DIAGRAM: Cron Verification System
Caption: Shows how each cron job is paired with a CueAPI verification task that checks outcomes.
Nodes:
1. Cron Daemon - fires ten jobs on their configured schedules
2. State Store (Postgres) - each job writes its outcome row on completion
3. CueAPI Scheduler - holds one verification task per cron job, offset by expected duration
4. Verification Agent (Python) - reads outcome from State Store, compares to expected state
5. Alert Channel (Slack webhook) - receives failure notifications
Flow:
- Cron Daemon fires Job N at scheduled time T
- Job N writes outcome row to State Store at time T+duration
- CueAPI fires Verification Agent at time T+duration+buffer
- Verification Agent queries State Store for Job N outcome
- If outcome missing or unexpected, Verification Agent posts to Alert Channel
```

## Step-by-step implementation

### Step 1: Create the state store table

Every cron job needs a place to record that it ran and what happened. A single Postgres table handles this. The table stores the job name, the expected run date, the actual completion timestamp, and a status column.

```sql
CREATE TABLE IF NOT EXISTS cron_outcomes (
  id SERIAL PRIMARY KEY,
  job_name TEXT NOT NULL,
  run_date DATE NOT NULL,
  completed_at TIMESTAMPTZ,
  status TEXT NOT NULL DEFAULT 'pending',
  error_message TEXT,
  UNIQUE(job_name, run_date)
);
```

### Step 2: Instrument your cron jobs to write outcomes

Each cron job must write a row when it finishes. This wrapper script runs any command and records the result. Replace your raw crontab entries with calls to this wrapper.

```bash
#!/usr/bin/env bash
# File: /usr/local/bin/cron_wrapper.sh
# Usage: cron_wrapper.sh <job_name> <command...>

JOB_NAME="$1"
shift
RUN_DATE=$(date +%F)

# Insert a pending row before execution
psql "$DATABASE_URL" -c \
  "INSERT INTO cron_outcomes (job_name, run_date, status) VALUES ('$JOB_NAME', '$RUN_DATE', 'running') ON CONFLICT (job_name, run_date) DO UPDATE SET status = 'running';"

# Run the actual job
if "$@"; then
  psql "$DATABASE_URL" -c \
    "UPDATE cron_outcomes SET status = 'success', completed_at = NOW() WHERE job_name = '$JOB_NAME' AND run_date = '$RUN_DATE';"
else
  EXIT_CODE=$?
  psql "$DATABASE_URL" -c \
    "UPDATE cron_outcomes SET status = 'failed', completed_at = NOW(), error_message = 'exit code $EXIT_CODE' WHERE job_name = '$JOB_NAME' AND run_date = '$RUN_DATE';"
fi
```

### Step 3: Define expected states in a config file

The verification agent needs to know what "correct" looks like for each job. A JSON config file maps job names to their expected schedule time, maximum allowed duration in minutes, and the status that counts as success.

```json
{
  "jobs": [
    {"name": "invoice-gen", "cron_utc": "06:00", "max_duration_min": 15, "expected_status": "success"},
    {"name": "inventory-sync", "cron_utc": "07:00", "max_duration_min": 30, "expected_status": "success"},
    {"name": "log-rotate", "cron_utc": "02:00", "max_duration_min": 5, "expected_status": "success"},
    {"name": "report-push", "cron_utc": "08:00", "max_duration_min": 20, "expected_status": "success"},
    {"name": "cache-warm", "cron_utc": "05:30", "max_duration_min": 10, "expected_status": "success"},
    {"name": "db-backup", "cron_utc": "03:00", "max_duration_min": 45, "expected_status": "success"},
    {"name": "email-digest", "cron_utc": "09:00", "max_duration_min": 10, "expected_status": "success"},
    {"name": "sftp-upload", "cron_utc": "10:00", "max_duration_min": 15, "expected_status": "success"},
    {"name": "metrics-agg", "cron_utc": "04:00", "max_duration_min": 20, "expected_status": "success"},
    {"name": "cert-renew", "cron_utc": "01:00", "max_duration_min": 5, "expected_status": "success"}
  ]
}
```

### Step 4: Get a CueAPI key

Sign up at [https://cueapi.com](https://cueapi.com) and create an API key from the dashboard at https://cueapi.com/dashboard/keys. Export it in your shell.

```bash
export CUEAPI_API_KEY="your-key-from-the-dashboard"
```

### Step 5: Write the verification agent

This Python script reads the config, queries the state store for each job's outcome for today, and returns a list of failures. It runs as a standalone function that CueAPI will invoke via a webhook.

```python
# File: verify_cron.py
import os
import json
import psycopg2
import requests
from datetime import date

DATABASE_URL = os.environ["DATABASE_URL"]
SLACK_WEBHOOK_URL = os.environ["SLACK_WEBHOOK_URL"]

def load_config():
    with open("cron_config.json") as f:
        return json.load(f)["jobs"]

def check_outcomes(job_list):
    conn = psycopg2.connect(DATABASE_URL)
    cur = conn.cursor()
    today = date.today().isoformat()
    failures = []
    for job in job_list:
        cur.execute(
            "SELECT status, error_message FROM cron_outcomes WHERE job_name = %s AND run_date = %s",
            (job["name"], today)
        )
        row = cur.fetchone()
        if row is None:
            failures.append({"job": job["name"], "issue": "no outcome row found, job may not have started"})
        elif row[0] != job["expected_status"]:
            failures.append({"job": job["name"], "issue": f"status is '{row[0]}', expected '{job['expected_status']}'", "error": row[1]})
    cur.close()
    conn.close()
    return failures

def alert(failures):
    if not failures:
        return
    blocks = ["*Cron Verification Failures*"]
    for f in failures:
        line = f"> `{f['job']}`: {f['issue']}"
        if f.get("error"):
            line += f" ({f['error']})"
        blocks.append(line)
    payload = {"text": "\n".join(blocks)}
    requests.post(SLACK_WEBHOOK_URL, json=payload, timeout=10)

if __name__ == "__main__":
    jobs = load_config()
    failures = check_outcomes(jobs)
    alert(failures)
    print(json.dumps({"checked": len(jobs), "failures": len(failures), "details": failures}, indent=2))
```

### Step 6: Expose the agent as a webhook endpoint

CueAPI calls a URL when a scheduled task fires. Wrap the verification logic in a small Flask app so CueAPI can trigger it.

```python
# File: server.py
import os
from flask import Flask, request, jsonify
from verify_cron import load_config, check_outcomes, alert

app = Flask(__name__)
CUEAPI_WEBHOOK_SECRET = os.environ.get("CUEAPI_WEBHOOK_SECRET", "")

@app.route("/verify", methods=["POST"])
def verify():
    token = request.headers.get("X-Cue-Secret", "")
    if token != CUEAPI_WEBHOOK_SECRET:
        return jsonify({"error": "unauthorized"}), 401
    jobs = load_config()
    failures = check_outcomes(jobs)
    alert(failures)
    return jsonify({"checked": len(jobs), "failures": len(failures), "details": failures})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8400)
```

### Step 7: Schedule verification tasks in CueAPI

For each job in the config, create a CueAPI scheduled task. The task fires at the job's cron time plus its max duration plus a five-minute buffer. This script reads the config and creates all ten tasks.

```python
# File: setup_cueapi_tasks.py
import os
import json
import requests

CUEAPI_API_KEY = os.environ["CUEAPI_API_KEY"]
CUEAPI_BASE = "https://api.cueapi.com/v1"
WEBHOOK_URL = os.environ["VERIFY_WEBHOOK_URL"]  # e.g. https://your-server.com/verify

def time_plus_minutes(time_str, minutes):
    h, m = map(int, time_str.split(":"))
    total = h * 60 + m + minutes
    return f"{(total // 60) % 24:02d}:{total % 60:02d}"

with open("cron_config.json") as f:
    jobs = json.load(f)["jobs"]

for job in jobs:
    fire_time = time_plus_minutes(job["cron_utc"], job["max_duration_min"] + 5)
    payload = {
        "name": f"verify-{job['name']}",
        "schedule": f"{fire_time} daily",
        "webhook_url": WEBHOOK_URL,
        "method": "POST",
        "headers": {"X-Cue-Secret": os.environ.get("CUEAPI_WEBHOOK_SECRET", "")},
        "body": json.dumps({"job": job["name"]})
    }
    resp = requests.post(
        f"{CUEAPI_BASE}/tasks",
        headers={"Authorization": f"Bearer {CUEAPI_API_KEY}", "Content-Type": "application/json"},
        json=payload,
        timeout=15
    )
    print(f"{job['name']}: {resp.status_code} {resp.text}")
```

### Step 8: Deploy and test with a forced failure

Deploy the Flask server. Then simulate a failure by inserting a bad outcome row and triggering the webhook manually.

```bash
# Insert a fake failure for today
psql "$DATABASE_URL" -c \
  "INSERT INTO cron_outcomes (job_name, run_date, status, error_message) VALUES ('invoice-gen', CURRENT_DATE, 'failed', 'exit code 1') ON CONFLICT (job_name, run_date) DO UPDATE SET status = 'failed', error_message = 'exit code 1';"

# Trigger the verification endpoint
curl -X POST https://your-server.com/verify \
  -H "X-Cue-Secret: $CUEAPI_WEBHOOK_SECRET" \
  -H "Content-Type: application/json"
```

You should see a Slack message within seconds naming `invoice-gen` as failed.

## Breakage

If you skip the verification step, you have instrumented cron jobs that write outcomes to a database nobody reads. The state store fills up with rows. Some say "success." Some say "failed." Some are missing entirely because a job never started. Nobody queries the table. You built a flight recorder with no one listening to the black box. The failure mode is identical to the before-state: silent, slow, discovered by a customer.

```text
DIAGRAM: Failure Without Verification
Caption: Shows the dead end when outcomes are recorded but never checked.
Nodes:
1. Cron Daemon - fires jobs on schedule
2. State Store (Postgres) - accumulates outcome rows
3. Nobody - no process reads the outcomes
4. Customer - discovers the failure days later
Flow:
- Cron Daemon fires Job N
- Job N writes outcome to State Store (including failures)
- No verification process reads the State Store
- Customer notices missing data and emails support
```

## The fix

The fix is exactly what Steps 5 through 7 build: a verification agent that CueAPI triggers on a schedule. The missing piece in the breakage scenario is the scheduled read of the state store paired with an alert. The `check_outcomes` function in `verify_cron.py` is that read. The CueAPI tasks are the schedule. The Slack webhook is the alert. Without all three, outcomes rot unread.

If you deployed Steps 1 through 4 but skipped 5 through 7, add them now. The code below is the core loop from `verify_cron.py`, repeated for emphasis. It runs against the same `cron_outcomes` table and `cron_config.json` from earlier steps.

```python
# The critical loop: read outcomes, compare to expected state, collect failures
failures = check_outcomes(load_config())
alert(failures)
```

That is two function calls. The first queries Postgres for each job's row. The second posts to Slack if any row is missing or unexpected. The agent does in under a second what a human does in thirty minutes of scanning dashboards, and the agent does not get distracted.

## Fixed state

```text
DIAGRAM: System With Verification Active
Caption: Shows the complete loop where every cron outcome is checked and failures are surfaced.
Nodes:
1. Cron Daemon - fires jobs on schedule
2. State Store (Postgres) - each job writes its outcome row
3. CueAPI Scheduler - triggers verification after each job's window
4. Verification Agent - reads State Store, compares to cron_config.json
5. Slack Channel - receives failure alerts within seconds
6. Operator - reads Slack, takes action same day
Flow:
- Cron Daemon fires Job N at time T
- Job N writes outcome to State Store
- CueAPI triggers Verification Agent at T + max_duration + 5 min
- Verification Agent queries State Store for Job N
- If outcome is missing or wrong, Verification Agent posts to Slack Channel
- Operator sees alert and investigates within minutes
```

## After

You have ten cron jobs. They run daily. When the invoice job fails on Tuesday, the verification agent fires fifteen minutes after the job's expected completion. It queries the `cron_outcomes` table, finds status "failed" with "exit code 1," and posts to Slack. You see the message before your second coffee. You fix the issue. No customer emails. No five-day gap. The agent checked all ten jobs in under a second. It will do the same thing tomorrow, and the day after, without forgetting, without getting pulled into a meeting, without deciding that checking cron logs can wait until after lunch.

## Takeaway

Every automated task needs an automated verifier. The pattern is: write an outcome, schedule a read of that outcome, alert on mismatch. Apply this to any fire-and-forget system where silence is ambiguous. Silence should mean success, not "nobody checked."
