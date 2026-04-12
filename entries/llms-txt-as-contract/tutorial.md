## Opening thesis

You will build, validate, and deploy an `llms.txt` file that gives any AI agent a structured map of your entire site. An agent that reads a well-structured `llms.txt` picks up your entire corpus in one pass, which a human writing documentation by hand would take months to make this legible. The file is small. The payoff is large. Your site stops being a fog of links and starts being a table of contents that machines can parse in seconds.

## Before

An AI crawler lands on your domain. It sees 400 pages. Some are blog posts from 2019. Some are deprecated API docs. Some are marketing pages with the word "platform" used 47 times. The crawler has no way to know which pages are the core product docs, which are changelogs, and which are noise. So it either indexes everything (burning tokens and producing garbage summaries) or picks pages at random. Your carefully written integration guide sits at `/docs/integrations/v3/setup` and the crawler never finds it. Meanwhile a competitor with 30 pages of clean docs gets perfectly represented in every AI answer. Your site is invisible not because the content is bad, but because no machine can tell what matters.

## Architecture

The system has three components: the `llms.txt` file itself, a validation script that checks its structure before deploy, and your existing static site or server. The file lives at the root of your domain, at `/.well-known/llms.txt` or `/llms.txt`. Agents look for it the way browsers look for `robots.txt`. The validation script runs in CI so broken files never ship.

```text
DIAGRAM: llms.txt serving pipeline
Caption: How an llms.txt file moves from authoring to agent consumption
Nodes:
1. llms.txt source file - Markdown file authored by site owner, lives in repo
2. Validation script - Python script that parses and rejects malformed files
3. CI pipeline - Runs validation on every push to main
4. Web server / CDN - Serves the file at /llms.txt with text/markdown content type
5. AI agent / crawler - Fetches /llms.txt as first request to understand the site
Flow:
- Author writes llms.txt source file and commits to repo
- CI pipeline runs validation script against llms.txt source file
- If validation passes, CI pipeline deploys to web server / CDN
- AI agent / crawler requests /llms.txt from web server / CDN
- AI agent / crawler uses the structured content to decide which pages to fetch next
```

## Step-by-step implementation

### 1. Create the llms.txt file

The `llms.txt` specification (documented at llmstxt.org) defines a simple markdown format. The file starts with an H1 containing your project or company name. Then a blockquote with a one-line description. Then sections with H2 headings that group your links. Each link is a markdown link on its own line, optionally followed by a colon and a short description. This is the entire contract. No YAML front matter. No JSON. Just markdown that both humans and machines read easily.

Create a file called `llms.txt` in your project root.

```markdown
# Acme API

> Acme API provides payment processing for SaaS platforms.

## Docs

- [Getting Started](https://acme.dev/docs/getting-started): Set up your first payment flow in 10 minutes
- [Authentication](https://acme.dev/docs/auth): API keys, OAuth, and webhook signatures
- [Webhooks](https://acme.dev/docs/webhooks): Events, retry logic, and payload schemas
- [Errors](https://acme.dev/docs/errors): Every error code with fix instructions

## API Reference

- [Payments](https://acme.dev/api/payments): Create, capture, and refund payments
- [Customers](https://acme.dev/api/customers): CRUD operations for customer records
- [Subscriptions](https://acme.dev/api/subscriptions): Billing cycles, trials, and upgrades

## SDKs

- [Python SDK](https://acme.dev/sdks/python): pip install acme-sdk
- [Node SDK](https://acme.dev/sdks/node): npm install @acme/sdk
- [Go SDK](https://acme.dev/sdks/go): go get acme.dev/sdk-go

## Changelog

- [Release Notes](https://acme.dev/changelog): Versioned changelog with migration guides
```

### 2. Create the full-content companion file

The spec also defines `llms-full.txt`, an optional companion that contains the actual content of your key pages concatenated into one file. This lets an agent ingest your entire corpus in a single HTTP request. Generate it by concatenating your core docs with clear section markers.

```bash
#!/bin/bash
# build-llms-full.sh
# Concatenates core doc pages into llms-full.txt

OUTPUT="llms-full.txt"
echo "# Acme API - Full Documentation" > "$OUTPUT"
echo "" >> "$OUTPUT"

for file in docs/getting-started.md docs/auth.md docs/webhooks.md docs/errors.md; do
  if [ -f "$file" ]; then
    echo "---" >> "$OUTPUT"
    echo "" >> "$OUTPUT"
    cat "$file" >> "$OUTPUT"
    echo "" >> "$OUTPUT"
  fi
done

echo "Generated $OUTPUT ($(wc -c < "$OUTPUT") bytes)"
```

### 3. Write a validation script

A malformed `llms.txt` is worse than no file at all. An agent that fetches a broken file may hallucinate structure or skip your site entirely. This Python script checks the required elements: one H1, a blockquote, at least one H2 section, and valid markdown links.

```python
#!/usr/bin/env python3
"""validate_llms_txt.py - Validate an llms.txt file against the spec."""
import re
import sys

def validate(path: str) -> list[str]:
    errors = []
    with open(path, "r", encoding="utf-8") as f:
        content = f.read()
        lines = content.strip().split("\n")

    # Check H1
    h1_lines = [l for l in lines if l.startswith("# ") and not l.startswith("## ")]
    if len(h1_lines) == 0:
        errors.append("Missing H1: file must start with a project name as H1.")
    if len(h1_lines) > 1:
        errors.append(f"Found {len(h1_lines)} H1 headings. Only one is allowed.")

    # Check blockquote
    blockquote_lines = [l for l in lines if l.startswith("> ")]
    if len(blockquote_lines) == 0:
        errors.append("Missing blockquote: add a one-line description after the H1.")

    # Check H2 sections
    h2_lines = [l for l in lines if l.startswith("## ")]
    if len(h2_lines) == 0:
        errors.append("No H2 sections found. Add at least one section grouping your links.")

    # Check links
    link_pattern = re.compile(r"\[.+\]\(https?://.+\)")
    links = [l for l in lines if link_pattern.search(l)]
    if len(links) == 0:
        errors.append("No markdown links found. The file needs links to your pages.")

    # Check for broken patterns
    for i, line in enumerate(lines, 1):
        if "example.com" in line:
            errors.append(f"Line {i}: contains example.com. Use real URLs.")

    return errors

if __name__ == "__main__":
    path = sys.argv[1] if len(sys.argv) > 1 else "llms.txt"
    errors = validate(path)
    if errors:
        print(f"FAIL: {len(errors)} error(s) in {path}")
        for e in errors:
            print(f"  - {e}")
        sys.exit(1)
    else:
        print(f"PASS: {path} is valid.")
        sys.exit(0)
```

### 4. Run validation locally

Run the validator against your file before committing. This catches mistakes early.

```bash
python3 validate_llms_txt.py llms.txt
```

### 5. Add validation to CI

Add a step to your CI config. This example uses GitHub Actions syntax. The job runs on every push and blocks the merge if the file is invalid.

```yaml
# .github/workflows/validate-llms-txt.yml
name: Validate llms.txt
on:
  push:
    paths:
      - "llms.txt"
      - "llms-full.txt"
  pull_request:
    paths:
      - "llms.txt"
      - "llms-full.txt"
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Validate llms.txt
        run: python3 validate_llms_txt.py llms.txt
```

### 6. Configure your server to serve the file with the correct content type

Agents expect `text/markdown` or `text/plain` as the content type. If your CDN serves it as `application/octet-stream`, some agents will skip it. This nginx snippet sets the correct header. For static hosts like Vercel or Netlify, use their header configuration files.

```nginx
# nginx.conf snippet
location = /llms.txt {
    default_type text/markdown;
    root /var/www/html;
}

location = /llms-full.txt {
    default_type text/markdown;
    root /var/www/html;
}
```

### 7. Add a Netlify or Vercel headers config as an alternative

If you deploy to Netlify, add a `_headers` file. For Vercel, add a `vercel.json` entry. Both ensure the correct MIME type.

```text
# _headers (Netlify)
/llms.txt
  Content-Type: text/markdown; charset=utf-8
/llms-full.txt
  Content-Type: text/markdown; charset=utf-8
```

### 8. Verify the deployed file is reachable

After deploy, confirm the file is accessible and has the right content type. This curl command checks both.

```bash
curl -sI https://acme.dev/llms.txt | grep -i content-type
# Expected: content-type: text/markdown; charset=utf-8

curl -s https://acme.dev/llms.txt | head -5
# Expected: your H1, blockquote, and first section
```

### 9. Test agent behavior with a simple fetch script

Simulate what an agent does when it finds your `llms.txt`. This script fetches the file, extracts all links, and prints a prioritized reading list. This is the same logic most LLM-based crawlers use internally.

```python
#!/usr/bin/env python3
"""simulate_agent.py - Fetch llms.txt and extract a reading plan."""
import re
import urllib.request

url = "https://acme.dev/llms.txt"
response = urllib.request.urlopen(url)
content = response.read().decode("utf-8")

links = re.findall(r"\[(.+?)\]\((https?://.+?)\)", content)

print(f"Agent reading plan from {url}:")
print(f"Found {len(links)} pages to index.\n")

for i, (title, href) in enumerate(links, 1):
    print(f"{i}. {title}: {href}")
```

## Breakage

Without validation in CI, the file drifts. Someone renames a doc page but forgets to update `llms.txt`. A merge conflict leaves a broken markdown link. The file ships with a URL pointing to a 404. An agent fetches the file, follows a dead link, gets nothing, and drops your site from its index. Worse, the agent may cache the broken state for days. You lose visibility not because you removed the file, but because you let it rot. The failure is silent. No monitoring fires. No user complains. You just stop appearing in AI-generated answers and have no idea why.

```text
DIAGRAM: Breakage scenario with stale llms.txt
Caption: What happens when llms.txt contains dead links and no validation catches it
Nodes:
1. llms.txt with stale links - Contains URLs that return 404
2. Web server - Serves the file without checking link health
3. AI agent - Fetches llms.txt, follows links, gets 404 errors
4. Agent index - Marks site as low-quality or drops it entirely
Flow:
- Developer updates docs but forgets to update llms.txt
- Web server serves stale llms.txt to AI agent
- AI agent follows dead links from llms.txt
- AI agent receives 404 responses and penalizes site in agent index
```

## The fix

Add a link-checking step to the validation script. This extends the `validate_llms_txt.py` file from step 3. The new function makes a HEAD request to every URL in the file and fails if any return a non-200 status. Run this in CI alongside the structural validation.

```python
#!/usr/bin/env python3
"""check_links.py - Verify all URLs in llms.txt are reachable."""
import re
import sys
import urllib.request
import urllib.error

def check_links(path: str) -> list[str]:
    errors = []
    with open(path, "r", encoding="utf-8") as f:
        content = f.read()

    links = re.findall(r"\[.+?\]\((https?://.+?)\)", content)

    for url in links:
        try:
            req = urllib.request.Request(url, method="HEAD")
            req.add_header("User-Agent", "llms-txt-validator/1.0")
            resp = urllib.request.urlopen(req, timeout=10)
            if resp.status != 200:
                errors.append(f"{url} returned status {resp.status}")
        except urllib.error.HTTPError as e:
            errors.append(f"{url} returned status {e.code}")
        except urllib.error.URLError as e:
            errors.append(f"{url} is unreachable: {e.reason}")

    return errors

if __name__ == "__main__":
    path = sys.argv[1] if len(sys.argv) > 1 else "llms.txt"
    errors = check_links(path)
    if errors:
        print(f"FAIL: {len(errors)} broken link(s) in {path}")
        for e in errors:
            print(f"  - {e}")
        sys.exit(1)
    else:
        print(f"PASS: All links in {path} are reachable.")
        sys.exit(0)
```

## Fixed state

```text
DIAGRAM: Validated llms.txt pipeline with link checking
Caption: The full pipeline with structural validation and link health checks
Nodes:
1. llms.txt source file - Markdown file authored by site owner
2. Structural validator - Checks H1, blockquote, H2 sections, and link format
3. Link checker - HEAD requests every URL and fails on non-200 responses
4. CI pipeline - Runs both validators, blocks deploy on failure
5. Web server / CDN - Serves validated file with correct content type
6. AI agent - Fetches llms.txt, follows links, indexes content successfully
Flow:
- Author commits llms.txt to repo
- CI pipeline runs structural validator against llms.txt
- CI pipeline runs link checker against llms.txt
- If both pass, CI pipeline deploys llms.txt to web server / CDN
- AI agent fetches llms.txt from web server / CDN
- AI agent follows every link, gets valid pages, indexes full corpus
```

## After

An AI crawler lands on your domain. It fetches `/llms.txt` as its first request. It reads one file and knows: this is a payment processing API, here are the four core docs in reading order, here is the API reference broken into three resources, here are three SDKs, and here is the changelog. The crawler indexes the 12 pages that matter. It skips the 388 pages that do not. It builds an accurate summary of your product in one pass. Your integration guide at `/docs/integrations/v3/setup` is listed right where you put it. You chose what the agent sees. You chose the order. The crawler reads `llms.txt`, gets a guided tour of your corpus, and indexes the pages that matter in the order you chose.

## Takeaway

The pattern is: give the machine a table of contents before it tries to read the book. This applies to any system where an automated consumer faces a large, unstructured corpus. One index file, validated in CI, with every link verified, turns a hundred scattered pages into a single readable contract. Build the map before the territory confuses the explorer.