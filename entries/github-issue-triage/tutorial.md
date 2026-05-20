## Opening thesis

You will build an agent that reads every open issue in a GitHub repository, classifies it by type and priority, and applies labels and a next-action comment. An agent reads, classifies, and labels a thousand issues per hour with deterministic rules, which a human maintainer gets through maybe thirty before burning out. The system runs once to clear the backlog, then hooks into a webhook to triage each new issue on arrival.

## Before

You open your repository's issue tracker and see four hundred open issues. No labels. No priorities. No assignment. Some are bug reports with stack traces. Some are feature requests disguised as questions. Some are duplicates of each other. A new contributor lands on the repo, clicks "Issues," and sees a wall of unsorted text. They close the tab. Your maintainers try to triage on Saturday mornings, but after thirty issues they are cooked. The backlog grows faster than anyone can read it. Every week, the same question in Discord: "Is anyone looking at issue 247?" Nobody knows.

## Architecture

The system has four components. A Python script fetches open issues from the GitHub REST API. It sends each issue's title and body to the Anthropic API, which returns a structured classification. The script parses that classification and applies labels and a comment via the GitHub API. A simple log file tracks every decision for auditing.

```text
DIAGRAM: Issue triage pipeline
Caption: Data flows from GitHub issues through Claude classification back to GitHub labels.
Nodes:
1. GitHub REST API - source of open issues, target for labels and comments
2. Fetcher (Python) - pulls issues in pages of 100, manages rate limits
3. Anthropic API (Claude) - classifies each issue into type, priority, next-action
4. Labeler (Python) - applies labels and posts a triage comment
5. triage_log.jsonl - append-only log of every classification decision
Flow:
- Fetcher pulls open issues from GitHub REST API (GET /repos/:owner/:repo/issues)
- Fetcher sends title + body to Anthropic API for classification
- Anthropic API returns JSON with type, priority, next_action
- Labeler writes labels to GitHub REST API (POST /repos/:owner/:repo/issues/:number/labels)
- Labeler posts triage comment to GitHub REST API (POST /repos/:owner/:repo/issues/:number/comments)
- Every decision is appended to triage_log.jsonl
```

## Step-by-step implementation

### 1. Set environment variables

You need two tokens. Get a GitHub personal access token at https://github.com/settings/tokens with the `repo` scope. Get an Anthropic API key at https://console.anthropic.com/settings/keys. Export both.

```bash
export GITHUB_TOKEN="ghp_your_token_here"
export ANTHROPIC_API_KEY="sk-ant-your_key_here"
export GITHUB_REPO="yourorg/yourrepo"
```

### 2. Install dependencies

The script uses two libraries: `requests` for HTTP and `anthropic` for the Claude API. Install them in a virtual environment.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install requests anthropic
```

### 3. Define the label taxonomy

Create a file called `taxonomy.py`. This is the single source of truth for your classification scheme. Every label listed here will be created in the repo if it does not exist. Deterministic rules start with a fixed vocabulary.

```python
# taxonomy.py

LABEL_COLORS = {
    "bug": "d73a4a",
    "feature-request": "0075ca",
    "question": "d876e3",
    "docs": "0e8a16",
    "duplicate": "cfd3d7",
    "stale": "ffffff",
}

PRIORITY_COLORS = {
    "p0-critical": "b60205",
    "p1-high": "d93f0b",
    "p2-medium": "fbca04",
    "p3-low": "c2e0c6",
}

NEXT_ACTIONS = [
    "needs-reproduction",
    "needs-design",
    "ready-to-fix",
    "close-as-duplicate",
    "close-as-stale",
    "needs-maintainer-input",
]

ALL_LABELS = {**LABEL_COLORS, **PRIORITY_COLORS}
```

### 4. Ensure labels exist in the repo

Before the agent can apply labels, they must exist. This script creates any missing labels. Run it once.

```python
# ensure_labels.py
import os
import requests
from taxonomy import ALL_LABELS

GITHUB_TOKEN = os.environ["GITHUB_TOKEN"]
REPO = os.environ["GITHUB_REPO"]
HEADERS = {"Authorization": f"token {GITHUB_TOKEN}", "Accept": "application/vnd.github+json"}

def ensure_labels():
    url = f"https://api.github.com/repos/{REPO}/labels"
    existing = []
    page = 1
    while True:
        resp = requests.get(url, headers=HEADERS, params={"per_page": 100, "page": page})
        resp.raise_for_status()
        batch = resp.json()
        if not batch:
            break
        existing.extend([l["name"] for l in batch])
        page += 1

    for name, color in ALL_LABELS.items():
        if name not in existing:
            requests.post(url, headers=HEADERS, json={"name": name, "color": color}).raise_for_status()
            print(f"Created label: {name}")

if __name__ == "__main__":
    ensure_labels()
```

### 5. Build the classifier prompt

Create `classifier.py`. The prompt instructs Claude to return valid JSON with three fields: `type`, `priority`, and `next_action`. The prompt pins the allowed values to the taxonomy. This is where deterministic rules live. Claude fills in the judgment; the schema constrains the output.

```python
# classifier.py
import os
import json
import anthropic
from taxonomy import LABEL_COLORS, PRIORITY_COLORS, NEXT_ACTIONS

CLIENT = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

SYSTEM_PROMPT = f"""You are a GitHub issue triage bot. Classify the issue below.
Return ONLY a JSON object with these three fields:
- "type": one of {json.dumps(list(LABEL_COLORS.keys()))}
- "priority": one of {json.dumps(list(PRIORITY_COLORS.keys()))}
- "next_action": one of {json.dumps(NEXT_ACTIONS)}

Rules:
1. If the issue contains a stack trace or error message, type is "bug".
2. If the issue asks "how do I" or "is it possible", type is "question".
3. If the issue proposes new behavior, type is "feature-request".
4. If the issue is about README, guides, or typos in text, type is "docs".
5. If the issue has had no activity for over 365 days and contains no clear action, type is "stale", priority is "p3-low", next_action is "close-as-stale".
6. Bugs with data loss or security implications are "p0-critical".
7. Bugs that block common workflows are "p1-high".
8. Everything else defaults to "p2-medium".
9. Questions and docs are "p3-low" unless they indicate a real gap.

Return raw JSON only. No markdown fences. No explanation."""

def classify_issue(title: str, body: str, created_at: str) -> dict:
    user_text = f"Title: {title}\nBody: {body or '(empty)'}\nCreated: {created_at}"
    message = CLIENT.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=256,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": user_text}],
    )
    raw = message.content[0].text.strip()
    return json.loads(raw)
```

### 6. Write the main triage loop

This is the core script. It fetches all open issues, classifies each one, applies labels, posts a comment, and logs the decision. Rate limiting is handled with a simple sleep. GitHub allows 5,000 requests per hour with a token. Each issue costs three API calls (fetch is amortized, plus one label call, one comment call). A thousand issues costs roughly 2,000 GitHub API calls and 1,000 Anthropic calls.

```python
# triage.py
import os
import json
import time
import requests
from classifier import classify_issue

GITHUB_TOKEN = os.environ["GITHUB_TOKEN"]
REPO = os.environ["GITHUB_REPO"]
HEADERS = {"Authorization": f"token {GITHUB_TOKEN}", "Accept": "application/vnd.github+json"}
LOG_FILE = "triage_log.jsonl"

def fetch_all_open_issues():
    issues = []
    page = 1
    while True:
        url = f"https://api.github.com/repos/{REPO}/issues"
        resp = requests.get(url, headers=HEADERS, params={"state": "open", "per_page": 100, "page": page})
        resp.raise_for_status()
        batch = resp.json()
        if not batch:
            break
        issues.extend([i for i in batch if "pull_request" not in i])
        page += 1
    return issues

def apply_labels(issue_number: int, labels: list):
    url = f"https://api.github.com/repos/{REPO}/issues/{issue_number}/labels"
    requests.post(url, headers=HEADERS, json={"labels": labels}).raise_for_status()

def post_comment(issue_number: int, body: str):
    url = f"https://api.github.com/repos/{REPO}/issues/{issue_number}/comments"
    requests.post(url, headers=HEADERS, json={"body": body}).raise_for_status()

def log_decision(entry: dict):
    with open(LOG_FILE, "a") as f:
        f.write(json.dumps(entry) + "\n")

def triage_all():
    issues = fetch_all_open_issues()
    print(f"Found {len(issues)} open issues.")
    for i, issue in enumerate(issues):
        number = issue["number"]
        title = issue["title"]
        body = issue.get("body", "") or ""
        created_at = issue["created_at"]

        try:
            result = classify_issue(title, body[:3000], created_at)
            labels = [result["type"], result["priority"]]
            apply_labels(number, labels)
            comment = (
                f"**Triage bot classification**\n\n"
                f"- Type: `{result['type']}`\n"
                f"- Priority: `{result['priority']}`\n"
                f"- Next action: `{result['next_action']}`\n\n"
                f"This was applied automatically. Maintainers: override by changing labels."
            )
            post_comment(number, comment)
            log_decision({"issue": number, "title": title, **result, "status": "ok"})
            print(f"[{i+1}/{len(issues)}] #{number}: {result['type']} / {result['priority']}")
        except Exception as e:
            log_decision({"issue": number, "title": title, "status": "error", "error": str(e)})
            print(f"[{i+1}/{len(issues)}] #{number}: ERROR: {e}")

        time.sleep(0.5)

if __name__ == "__main__":
    triage_all()
```

### 7. Run the backlog triage

Execute the script. On a repo with four hundred issues, expect it to finish in roughly twenty minutes. The Anthropic API handles the classification in under a second per issue. The sleep keeps you well within GitHub's rate limit.

```bash
python triage.py
```

### 8. Set up ongoing triage with a cron job

After the backlog is clear, run the script on a schedule to catch new issues. A cron job every fifteen minutes is enough. Alternatively, trigger it from a GitHub Actions workflow on the `issues: opened` event.

```bash
# Add to crontab: run every 15 minutes
*/15 * * * * cd /path/to/triage && source .venv/bin/activate && python triage.py >> triage_cron.log 2>&1
```

### 9. Skip already-triaged issues

The triage loop should not re-label issues that already have labels. Add a filter at the top of the loop. This makes the script idempotent.

```python
# Add this check at the start of the for loop in triage_all(), before classify_issue()
if issue.get("labels") and len(issue["labels"]) > 0:
    print(f"[{i+1}/{len(issues)}] #{number}: already labeled, skipping")
    continue
```

## Breakage

If you skip the audit log, you have no way to verify the agent's accuracy. A misclassified issue gets the wrong label, the wrong priority, and the wrong next-action. Nobody notices because there is no record of what the agent decided or why. A bug labeled "question" sits for months. A duplicate stays open and collects confused comments. The agent becomes a source of noise instead of signal. Without the log, you cannot measure accuracy, cannot spot systematic errors, and cannot improve the prompt. You have automated the production of garbage.

```text
DIAGRAM: Failure mode without audit log
Caption: Without logging, misclassifications are invisible and uncorrectable.
Nodes:
1. GitHub REST API - issues are labeled, but no record of why
2. Anthropic API - returns classification, but output is discarded after use
3. Labeler - applies labels with no accountability
4. (missing) triage_log.jsonl - does not exist
Flow:
- Labeler applies labels to GitHub
- Classification reasoning is lost
- Maintainer sees wrong label, has no way to trace the decision
- Systematic prompt errors go undetected
```

## The fix

The audit log is already present in the triage script above (`log_decision` writes to `triage_log.jsonl`). The fix is a verification script that reads the log and produces an accuracy report. A maintainer reviews a random sample of twenty issues and marks each classification as correct or incorrect. The script computes accuracy by type and priority. If accuracy drops below 90%, you know the prompt needs revision. This closes the feedback loop.

```python
# verify.py
import json
import random

LOG_FILE = "triage_log.jsonl"

def load_log():
    entries = []
    with open(LOG_FILE) as f:
        for line in f:
            entry = json.loads(line)
            if entry.get("status") == "ok":
                entries.append(entry)
    return entries

def sample_and_review(n=20):
    entries = load_log()
    sample = random.sample(entries, min(n, len(entries)))
    correct = 0
    for entry in sample:
        print(f"\nIssue #{entry['issue']}: {entry['title']}")
        print(f"  Type: {entry['type']}  Priority: {entry['priority']}  Action: {entry['next_action']}")
        answer = input("  Correct? (y/n): ").strip().lower()
        if answer == "y":
            correct += 1
    total = len(sample)
    accuracy = (correct / total) * 100 if total > 0 else 0
    print(f"\nAccuracy: {correct}/{total} = {accuracy:.1f}%")
    if accuracy < 90:
        print("Below 90%. Revise the classifier prompt in classifier.py.")
    else:
        print("Above 90%. Prompt is performing well.")

if __name__ == "__main__":
    sample_and_review()
```

## Fixed state

```text
DIAGRAM: Complete triage pipeline with verification
Caption: The audit log feeds a verification step that measures accuracy and triggers prompt revision.
Nodes:
1. GitHub REST API - source and target for issues and labels
2. Fetcher (Python) - pulls open issues, skips already-labeled ones
3. Anthropic API (Claude) - classifies each issue with constrained JSON output
4. Labeler (Python) - applies labels and posts triage comment
5. triage_log.jsonl - append-only decision log
6. verify.py - samples log entries, computes accuracy, flags prompt drift
Flow:
- Fetcher pulls unlabeled issues from GitHub REST API
- Fetcher sends title + body to Anthropic API
- Anthropic API returns structured classification
- Labeler applies labels and comments to GitHub REST API
- Every decision is appended to triage_log.jsonl
- verify.py reads triage_log.jsonl and produces accuracy report
- If accuracy drops below 90%, maintainer revises SYSTEM_PROMPT in classifier.py
```

## After

You open your repository's issue tracker and see four hundred issues, each with a colored label and a priority. Bug reports have `bug` and `p1-high`. Feature requests have `feature-request` and `p2-medium`. Stale issues from 2023 are flagged `stale` with a suggestion to close. A new contributor lands on the repo, clicks "Issues," filters by `bug` and `p1-high`, and finds a task they can start on today. Your maintainers spend Saturday morning reviewing the agent's triage comments, overriding five or six misclassifications out of four hundred. The audit log shows 94% accuracy. The backlog is not a wall of noise. It is a sorted queue.

## Takeaway

The pattern is: constrain the output, log every decision, verify a sample. Classification tasks that follow a fixed taxonomy are ideal for agents because the rules are expressible in a prompt and the output is validatable against a schema. Apply this pattern to any backlog where humans burn out on repetitive reading: support tickets, pull request reviews, dependency alerts.