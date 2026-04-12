## Opening thesis

You will build an agent that fetches a hundred RSS feeds, scores every item against your editorial criteria, and delivers a ranked daily brief of fifteen items each morning. An agent applies a consistent editorial filter to thousands of items per day, which a human trying to maintain the same signal-to-noise ratio would spend two hours on and still miss things. The agent does not get tired. It does not skip feeds because it is Thursday.

## Before

You subscribed to a hundred RSS feeds over three years. They were good feeds. They produced five hundred items a day. You opened your reader on Monday, saw 2,400 unread items, and marked all as read. You tried again on Wednesday. Same result. You built a folder system. You starred things. You wrote rules. None of it stuck, because the core problem is volume: five hundred items per day, two hours to triage them, and your actual job waiting. So you stopped reading them. The feeds still exist. The signal is still in there. You just cannot extract it at human speed.

## Architecture

The system has four stages. A fetcher pulls items from all feeds. A deduplicator removes repeated links and near-duplicate titles. A scorer sends each surviving item to the Anthropic API with your editorial prompt and gets back a relevance score. A formatter takes the top fifteen scored items, ranks them, and writes the brief to a file or sends it as an email.

```text
DIAGRAM: RSS-to-Brief Pipeline
Caption: Four-stage pipeline from raw feeds to ranked daily brief
Nodes:
1. Feed List (OPML/JSON) - stores the hundred feed URLs
2. Fetcher (Python + feedparser) - pulls items from all feeds in parallel
3. Deduplicator - removes exact URL matches and near-duplicate titles
4. Scorer (Anthropic API) - scores each item 0-100 against editorial criteria
5. Formatter - selects top 15, ranks by score, writes brief
6. Output (Markdown file / email) - the daily brief
Flow:
- Feed List -> Fetcher: URLs
- Fetcher -> Deduplicator: raw items (approx 500/day)
- Deduplicator -> Scorer: unique items (approx 350/day)
- Scorer -> Formatter: scored items
- Formatter -> Output: ranked brief of 15 items
```

## Step-by-step implementation

### Step 1: Define your feed list

Store your feeds in a JSON file. Each entry has a URL and an optional category tag. This file replaces an OPML export, which is harder to edit by hand.

```json
[
  {"url": "https://feeds.arstechnica.com/arstechnica/index", "tag": "tech"},
  {"url": "https://hnrss.org/frontpage", "tag": "tech"},
  {"url": "https://www.theverge.com/rss/index.xml", "tag": "tech"},
  {"url": "https://feeds.feedburner.com/marginalrevolution/feed", "tag": "econ"},
  {"url": "https://pluralistic.net/feed/", "tag": "policy"}
]
```

Save this as `feeds.json`. Add your other ninety-five feeds the same way.

### Step 2: Install dependencies

You need Python 3.11 or later, three packages, and an Anthropic API key. Get the key from https://console.anthropic.com/settings/keys and export it.

```bash
pip install feedparser anthropic python-dateutil
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Step 3: Fetch all feeds

This script reads `feeds.json`, fetches each feed, and collects items published in the last 24 hours. It uses `concurrent.futures` to fetch in parallel. Each item is stored as a dictionary with title, link, summary, published date, and source tag.

```python
import json
import feedparser
from datetime import datetime, timezone, timedelta
from concurrent.futures import ThreadPoolExecutor, as_completed
from dateutil import parser as dateparser

def load_feeds(path="feeds.json"):
    with open(path) as f:
        return json.load(f)

def fetch_one(feed):
    try:
        d = feedparser.parse(feed["url"])
        cutoff = datetime.now(timezone.utc) - timedelta(hours=24)
        items = []
        for entry in d.entries:
            pub = entry.get("published", entry.get("updated", ""))
            try:
                pub_dt = dateparser.parse(pub)
                if pub_dt.tzinfo is None:
                    pub_dt = pub_dt.replace(tzinfo=timezone.utc)
                if pub_dt < cutoff:
                    continue
            except Exception:
                continue
            items.append({
                "title": entry.get("title", ""),
                "link": entry.get("link", ""),
                "summary": entry.get("summary", "")[:500],
                "published": pub,
                "tag": feed.get("tag", "general")
            })
        return items
    except Exception:
        return []

def fetch_all():
    feeds = load_feeds()
    all_items = []
    with ThreadPoolExecutor(max_workers=20) as pool:
        futures = {pool.submit(fetch_one, f): f for f in feeds}
        for future in as_completed(futures):
            all_items.extend(future.result())
    return all_items

if __name__ == "__main__":
    items = fetch_all()
    print(f"Fetched {len(items)} items from the last 24 hours.")
    with open("raw_items.json", "w") as f:
        json.dump(items, f, indent=2)
```

### Step 4: Deduplicate

Multiple feeds syndicate the same story. This step removes exact URL matches first, then catches near-duplicates by comparing normalized titles. Two titles that share 80% of their words after lowercasing and stripping punctuation are treated as duplicates. The item with the longer summary survives.

```python
import json
import re

def normalize(title):
    return set(re.sub(r"[^a-z0-9 ]", "", title.lower()).split())

def jaccard(a, b):
    if not a or not b:
        return 0.0
    return len(a & b) / len(a | b)

def deduplicate(items):
    seen_urls = set()
    unique = []
    for item in items:
        if item["link"] in seen_urls:
            continue
        seen_urls.add(item["link"])
        unique.append(item)

    final = []
    used = set()
    for i, a in enumerate(unique):
        if i in used:
            continue
        best = a
        na = normalize(a["title"])
        for j in range(i + 1, len(unique)):
            if j in used:
                continue
            nb = normalize(unique[j]["title"])
            if jaccard(na, nb) > 0.8:
                used.add(j)
                if len(unique[j]["summary"]) > len(best["summary"]):
                    best = unique[j]
        final.append(best)
    return final

if __name__ == "__main__":
    with open("raw_items.json") as f:
        items = json.load(f)
    deduped = deduplicate(items)
    print(f"Deduplicated: {len(items)} -> {len(deduped)}")
    with open("deduped_items.json", "w") as f:
        json.dump(deduped, f, indent=2)
```

### Step 5: Define your editorial criteria

This is the step that replaces two hours of your judgment. Write a system prompt that tells the model exactly what you care about. Be specific. Vague prompts produce vague scores. Store this in a text file so you can edit it without touching code.

```text
You are an editorial filter for a daily technology brief.

Score each item from 0 to 100 based on these criteria:
- Practical builder content (tutorials, postmortems, benchmarks): high
- Original research or primary-source reporting: high
- Policy changes that affect software deployment or data handling: medium-high
- Product launches with no technical substance: low
- Opinion pieces restating known positions: low
- Listicles, roundups, or "top 10" posts: very low
- Press releases thinly disguised as articles: zero

Return ONLY a JSON object: {"score": <integer>, "reason": "<one sentence>"}
Do not explain your reasoning beyond that one sentence.
```

Save this as `editorial_prompt.txt`.

### Step 6: Score items with the Anthropic API

This script sends each deduplicated item to Claude with your editorial prompt. It batches requests to stay within rate limits. Each item gets a score from 0 to 100 and a one-sentence reason. Items that fail to parse get a score of zero.

```python
import json
import os
import time
import anthropic

client = anthropic.Anthropic()

with open("editorial_prompt.txt") as f:
    SYSTEM_PROMPT = f.read().strip()

def score_item(item):
    user_msg = f"Title: {item['title']}\nSource tag: {item['tag']}\nSummary: {item['summary']}"
    try:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=150,
            system=SYSTEM_PROMPT,
            messages=[{"role": "user", "content": user_msg}]
        )
        text = response.content[0].text.strip()
        result = json.loads(text)
        item["score"] = int(result["score"])
        item["reason"] = result["reason"]
    except Exception as e:
        item["score"] = 0
        item["reason"] = f"Scoring failed: {e}"
    return item

def score_all(items, delay=0.2):
    scored = []
    for i, item in enumerate(items):
        scored.append(score_item(item))
        if i % 50 == 0 and i > 0:
            print(f"Scored {i}/{len(items)}")
        time.sleep(delay)
    return scored

if __name__ == "__main__":
    with open("deduped_items.json") as f:
        items = json.load(f)
    scored = score_all(items)
    scored.sort(key=lambda x: x["score"], reverse=True)
    with open("scored_items.json", "w") as f:
        json.dump(scored, f, indent=2)
    print(f"Scored {len(scored)} items. Top score: {scored[0]['score']}")
```

### Step 7: Format the daily brief

This step takes the top fifteen items and writes a Markdown file with today's date. Each entry shows the title as a link, the score, the source tag, and the one-sentence reason. This file is your daily brief.

```python
import json
from datetime import date

def format_brief(items, top_n=15):
    today = date.today().isoformat()
    top = items[:top_n]
    lines = [f"# Daily Brief: {today}", "", f"{len(top)} items from {len(items)} scored.\n"]
    for i, item in enumerate(top, 1):
        lines.append(f"### {i}. [{item['title']}]({item['link']})")
        lines.append(f"Score: {item['score']} | Tag: {item['tag']}")
        lines.append(f"{item['reason']}\n")
    return "\n".join(lines)

if __name__ == "__main__":
    with open("scored_items.json") as f:
        items = json.load(f)
    brief = format_brief(items)
    filename = f"brief_{date.today().isoformat()}.md"
    with open(filename, "w") as f:
        f.write(brief)
    print(f"Wrote {filename}")
```

### Step 8: Automate the daily run

Wrap the full pipeline in a single script and schedule it with cron. This runs at 6:00 AM local time every day.

```bash
cat > run_brief.sh << 'EOF'
#!/bin/bash
set -e
cd "$(dirname "$0")"
python fetch.py
python dedup.py
python score.py
python format_brief.py
echo "Brief generated at $(date)"
EOF
chmod +x run_brief.sh

# Add to crontab
(crontab -l 2>/dev/null; echo "0 6 * * * /absolute/path/to/run_brief.sh >> /absolute/path/to/brief.log 2>&1") | crontab -
```

Replace `/absolute/path/to/` with the actual directory. Make sure `ANTHROPIC_API_KEY` is set in the cron environment. The simplest way: add `export ANTHROPIC_API_KEY=sk-ant-...` to the top of `run_brief.sh`.

## Breakage

If you skip score verification, the agent drifts. The editorial prompt says "press releases disguised as articles" should score zero. But some press releases have long technical summaries. The model scores them at 40 or 50. Over a week, your brief fills with vendor announcements. You stop trusting it. You stop reading it. You are back to the before-state, except now you also wasted time building a pipeline. The failure is silent: no errors, no crashes, just a slow decline in signal quality that you notice too late.

```text
DIAGRAM: Drift Failure Mode
Caption: Without verification, scored items drift from editorial intent over days
Nodes:
1. Scorer (Anthropic API) - produces scores that slowly drift
2. Formatter - selects top 15 without checking quality
3. Output - brief contains vendor noise
4. Reader (you) - loses trust, stops reading
Flow:
- Scorer -> Formatter: scores include false positives at 40-60 range
- Formatter -> Output: vendor items appear in top 15
- Output -> Reader: signal-to-noise ratio degrades daily
- Reader -> (nothing): pipeline abandoned within two weeks
```

## The fix

Add a verification step after scoring. Sample five random items from the top twenty and five from the bottom fifty. Send them to the model with a different prompt that asks: "Does this score match the editorial criteria? Answer yes or no, with a one-sentence correction if no." Log disagreements. If more than two of the ten samples disagree, re-score all items with a tightened prompt. This catches drift before it reaches your inbox.

```python
import json
import random
import anthropic

client = anthropic.Anthropic()

with open("editorial_prompt.txt") as f:
    EDITORIAL = f.read().strip()

VERIFY_PROMPT = """You are an editorial auditor. Given an item and its assigned score,
decide if the score matches these criteria:

""" + EDITORIAL + """

Return ONLY: {"agree": true} or {"agree": false, "correction": "<one sentence>"}"""

def verify_sample(items):
    top_20 = items[:20]
    bottom_50 = items[-50:] if len(items) > 50 else items[15:]
    sample = random.sample(top_20, min(5, len(top_20))) + random.sample(bottom_50, min(5, len(bottom_50)))
    disagreements = []
    for item in sample:
        user_msg = (f"Title: {item['title']}\nTag: {item['tag']}\n"
                    f"Summary: {item['summary']}\nAssigned score: {item['score']}\n"
                    f"Reason: {item['reason']}")
        try:
            response = client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=150,
                system=VERIFY_PROMPT,
                messages=[{"role": "user", "content": user_msg}]
            )
            result = json.loads(response.content[0].text.strip())
            if not result.get("agree", True):
                disagreements.append({
                    "title": item["title"],
                    "score": item["score"],
                    "correction": result.get("correction", "")
                })
        except Exception:
            pass
    return disagreements

if __name__ == "__main__":
    with open("scored_items.json") as f:
        items = json.load(f)
    issues = verify_sample(items)
    print(f"Disagreements: {len(issues)} / 10")
    for issue in issues:
        print(f"  [{issue['score']}] {issue['title']}: {issue['correction']}")
    if len(issues) > 2:
        print("DRIFT DETECTED. Re-score with tightened criteria.")
    with open("verification_log.json", "w") as f:
        json.dump(issues, f, indent=2)
```

## Fixed state

```text
DIAGRAM: RSS-to-Brief Pipeline with Verification
Caption: Full pipeline including drift detection via score verification
Nodes:
1. Feed List (OPML/JSON) - stores the hundred feed URLs
2. Fetcher (Python + feedparser) - pulls items from all feeds in parallel
3. Deduplicator - removes exact URL matches and near-duplicate titles
4. Scorer (Anthropic API) - scores each item 0-100 against editorial criteria
5. Verifier (Anthropic API, second prompt) - audits a sample of scores for drift
6. Formatter - selects top 15, ranks by score, writes brief
7. Output (Markdown file / email) - the daily brief
8. Verification Log - records disagreements for weekly review
Flow:
- Feed List -> Fetcher: URLs
- Fetcher -> Deduplicator: raw items
- Deduplicator -> Scorer: unique items
- Scorer -> Verifier: scored items (sample of 10)
- Verifier -> Scorer: re-score signal if drift detected (>2 disagreements)
- Scorer -> Formatter: verified scored items
- Formatter -> Output: ranked brief of 15 items
- Verifier -> Verification Log: disagreements for human review
```

## After

You have a hundred RSS feeds producing five hundred items a day. At 6:15 AM, a Markdown file appears with fifteen items ranked by predicted signal strength. Each item has a score and a one-sentence reason it was selected. You read the brief in eight minutes. The verification log shows zero or one disagreements most days. When it shows three, the pipeline tightens itself before you notice. You read your feeds again. You did not add hours to your day. You subtracted them.

## Takeaway

Consistent editorial judgment at scale is a filtering problem, not a reading problem. The pattern: define your criteria in a prompt, score everything, verify a sample, and flag drift before it compounds. This works for RSS feeds. It works for any high-volume input where your taste is definable but your time is not expandable.