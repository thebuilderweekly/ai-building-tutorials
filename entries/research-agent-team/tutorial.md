## Opening thesis

You will build a three-agent research team where each agent has one narrow job and hands typed JSON to the next agent in line. The Researcher finds primary sources. The Verifier checks every claim against those sources. The Summarizer writes the final brief using only verified material. Three specialized agents working in sequence produce verified research in five minutes; one generalist agent doing all three jobs takes twenty minutes and produces ungrounded claims. By the end, your team will refuse to write any sentence it cannot point to a source for.

## Before

You ask Claude to research the impact of GLP-1 drugs on the restaurant industry. Five minutes later you have a 400-word summary. It cites declining alcohol sales at chain restaurants. It mentions a 12% drop in dessert orders. It quotes an executive at a fast-casual chain. The summary reads like real journalism. Three of those facts are wrong. The 12% number is a hallucination. The executive quote is paraphrased from a different industry. Only one detail traces back to a real source. You sent the summary to your investment partner before you checked. Now you are explaining why your "research" included a fabricated quote. Your partner stops trusting the agent. You stop using the agent. You go back to manual research, which takes hours per topic. The agent was fast, but speed without grounding is just sophisticated guessing.

## Architecture

The system has three agents and one shared context object. The Researcher reads the topic, queries an external search API, and returns a list of source documents with URLs and excerpts. The Verifier reads draft claims and checks each one against the source documents, returning a list of verified, partially-verified, and rejected claims. The Summarizer reads the verified claims and writes the final brief, refusing to add any sentence not backed by a verified claim.

```text
DIAGRAM: Three-agent research team
Caption: Sequential agents with typed JSON handoffs and explicit verification before final write
Nodes:
1. Topic input - The research question from the user
2. Researcher agent (Claude) - Generates search queries and fetches primary sources
3. Search API (Brave) - Returns ranked results with URLs and excerpts
4. Sources collection - Typed list of {url, title, excerpt, fetched_at}
5. Drafter (Claude) - Produces draft claims that need verification
6. Claims collection - Typed list of {claim, suggested_source_url}
7. Verifier agent (Claude) - Checks each claim against the source it cites
8. Verified claims - Typed list of {claim, status, source_url, evidence}
9. Summarizer agent (Claude) - Writes final brief using only verified claims
10. Final brief - The output the user receives
Flow:
- Topic input goes to Researcher agent
- Researcher agent calls Search API for each query it generates
- Search API returns results, Researcher agent assembles Sources collection
- Sources collection goes to Drafter
- Drafter produces Claims collection by extracting candidate facts from sources
- Claims collection plus Sources collection go to Verifier agent
- Verifier agent reads each claim, fetches the cited source excerpt, returns Verified claims
- Verified claims go to Summarizer agent
- Summarizer agent writes Final brief using only claims marked verified
```

## Step-by-step implementation

### Step 1: Install dependencies

You need the Anthropic SDK and a search API client. Brave Search has a free tier that works for prototyping. Sign up at https://brave.com/search/api/ and copy your subscription token from the dashboard.

```bash
pip install anthropic requests
export ANTHROPIC_API_KEY="sk-ant-..."
export BRAVE_API_KEY="BSA..."
```

### Step 2: Define the typed contract between agents

Each agent reads and writes JSON objects with explicit shapes. This contract is what makes specialization possible: every agent knows exactly what it receives and what it must return. Save this as `contracts.py` and import it in every agent module.

```python
# contracts.py
from dataclasses import dataclass, asdict
from typing import Literal
import json

@dataclass
class Source:
    url: str
    title: str
    excerpt: str
    fetched_at: str

@dataclass
class Claim:
    text: str
    suggested_source_url: str

@dataclass
class VerifiedClaim:
    text: str
    status: Literal["verified", "partial", "rejected"]
    source_url: str
    evidence: str

def to_json(obj) -> str:
    return json.dumps(asdict(obj) if hasattr(obj, '__dataclass_fields__') else obj, indent=2)
```

### Step 3: Build the Researcher agent

The Researcher has one job: turn a topic into source documents. It generates 3-5 search queries, calls the Brave API, fetches the top results for each query, and assembles them into the Sources collection. It does not summarize. It does not interpret. It collects.

```python
# researcher.py
import os
import json
import requests
from datetime import datetime
import anthropic
from contracts import Source

client = anthropic.Anthropic()

RESEARCHER_PROMPT = """You are a research agent. Your only job is to generate 3-5 specific search queries that will find primary sources about the given topic.

Rules:
- Each query should be 3-8 words.
- Prefer queries that target original reporting, data sources, or primary documents.
- Avoid queries that target opinion pieces or aggregator summaries.
- Return ONLY a JSON array of strings: ["query 1", "query 2", ...]
- No markdown, no commentary."""

def generate_queries(topic: str) -> list[str]:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=300,
        system=RESEARCHER_PROMPT,
        messages=[{"role": "user", "content": f"Topic: {topic}"}],
    )
    text = response.content[0].text.strip()
    if text.startswith("```"):
        text = text.split("\n", 1)[1].rsplit("```", 1)[0].strip()
    return json.loads(text)

def search_brave(query: str, count: int = 3) -> list[dict]:
    response = requests.get(
        "https://api.search.brave.com/res/v1/web/search",
        headers={"X-Subscription-Token": os.environ["BRAVE_API_KEY"]},
        params={"q": query, "count": count},
        timeout=10,
    )
    response.raise_for_status()
    return response.json().get("web", {}).get("results", [])

def collect_sources(topic: str) -> list[Source]:
    queries = generate_queries(topic)
    sources = []
    seen_urls = set()
    now = datetime.utcnow().isoformat() + "Z"
    for query in queries:
        results = search_brave(query)
        for r in results:
            url = r.get("url", "")
            if url in seen_urls or not url:
                continue
            seen_urls.add(url)
            sources.append(Source(
                url=url,
                title=r.get("title", ""),
                excerpt=r.get("description", "")[:1000],
                fetched_at=now,
            ))
    return sources
```

### Step 4: Build the Drafter

The Drafter takes the Sources collection and extracts candidate claims. Each claim cites the source URL it came from. The Drafter is greedy on purpose: it produces every claim it can find, even ones that look weak. The Verifier filters them in the next step.

```python
# drafter.py
import json
import anthropic
from contracts import Source, Claim

client = anthropic.Anthropic()

DRAFTER_PROMPT = """You extract factual claims from source documents.

Rules:
- Each claim must be one declarative sentence.
- Each claim must cite the source URL it came from.
- Extract every distinct claim you find. Do not filter for importance yet.
- Do not invent claims that are not directly stated in the sources.
- Return ONLY a JSON array: [{"text": "...", "suggested_source_url": "..."}, ...]"""

def draft_claims(topic: str, sources: list[Source]) -> list[Claim]:
    sources_block = "\n\n".join(
        f"SOURCE_URL: {s.url}\nTITLE: {s.title}\nEXCERPT: {s.excerpt}"
        for s in sources
    )
    user_msg = f"Topic: {topic}\n\nSources:\n\n{sources_block}"
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2000,
        system=DRAFTER_PROMPT,
        messages=[{"role": "user", "content": user_msg}],
    )
    text = response.content[0].text.strip()
    if text.startswith("```"):
        text = text.split("\n", 1)[1].rsplit("```", 1)[0].strip()
    raw = json.loads(text)
    return [Claim(text=c["text"], suggested_source_url=c["suggested_source_url"]) for c in raw]
```

### Step 5: Build the Verifier agent

The Verifier is the trust layer. For each claim, it reads the cited source's excerpt and decides one of three outcomes: verified (the source clearly supports the claim), partial (the source touches the topic but does not fully support the claim), or rejected (the source does not support the claim or contradicts it). The Verifier returns the supporting evidence text alongside its verdict so a human can audit.

```python
# verifier.py
import json
import anthropic
from contracts import Source, Claim, VerifiedClaim

client = anthropic.Anthropic()

VERIFIER_PROMPT = """You verify whether a claim is supported by a source excerpt.

For the given claim and source excerpt, return one of three verdicts:
- "verified": The source excerpt directly supports the claim.
- "partial": The source mentions the topic but does not fully support the claim as stated.
- "rejected": The source does not support the claim, or contradicts it.

Quote the specific sentence or phrase from the excerpt that justifies your verdict. If rejected, quote the closest sentence that shows the gap.

Return ONLY a JSON object: {"status": "verified|partial|rejected", "evidence": "..."}"""

def verify_one(claim: Claim, source: Source) -> VerifiedClaim:
    user_msg = (
        f"Claim: {claim.text}\n\n"
        f"Source URL: {source.url}\n"
        f"Source excerpt: {source.excerpt}"
    )
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=400,
        system=VERIFIER_PROMPT,
        messages=[{"role": "user", "content": user_msg}],
    )
    text = response.content[0].text.strip()
    if text.startswith("```"):
        text = text.split("\n", 1)[1].rsplit("```", 1)[0].strip()
    result = json.loads(text)
    return VerifiedClaim(
        text=claim.text,
        status=result["status"],
        source_url=source.url,
        evidence=result["evidence"],
    )

def verify_all(claims: list[Claim], sources: list[Source]) -> list[VerifiedClaim]:
    source_map = {s.url: s for s in sources}
    verified = []
    for claim in claims:
        source = source_map.get(claim.suggested_source_url)
        if source is None:
            verified.append(VerifiedClaim(
                text=claim.text,
                status="rejected",
                source_url=claim.suggested_source_url,
                evidence="Source URL not found in collected sources.",
            ))
            continue
        verified.append(verify_one(claim, source))
    return verified
```

### Step 6: Build the Summarizer agent

The Summarizer reads only the verified claims (status equals "verified") and writes the final brief. Partial and rejected claims are excluded entirely. The Summarizer is forbidden from adding any sentence that is not backed by a verified claim. This is the constraint that makes the team's output trustworthy.

```python
# summarizer.py
import anthropic
from contracts import VerifiedClaim

client = anthropic.Anthropic()

SUMMARIZER_PROMPT = """You write a research brief using ONLY the verified claims provided.

Rules:
- Every sentence in your brief must correspond to a verified claim.
- Do not add background, context, or framing that is not in the claims.
- Do not soften or strengthen the claims. Use them as written.
- Cite each claim with [n] where n is the claim's index in the input list.
- Length: 200-400 words.
- If fewer than three claims are verified, return: "Insufficient verified claims to produce a brief."

Return only the brief text. No commentary, no markdown headers."""

def summarize(topic: str, verified_claims: list[VerifiedClaim]) -> str:
    accepted = [c for c in verified_claims if c.status == "verified"]
    if len(accepted) < 3:
        return "Insufficient verified claims to produce a brief."
    claims_block = "\n".join(
        f"[{i+1}] {c.text} (source: {c.source_url})"
        for i, c in enumerate(accepted)
    )
    user_msg = f"Topic: {topic}\n\nVerified claims:\n{claims_block}"
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1500,
        system=SUMMARIZER_PROMPT,
        messages=[{"role": "user", "content": user_msg}],
    )
    return response.content[0].text.strip()
```

### Step 7: Wire the team together

The orchestrator runs the four steps in sequence: collect sources, draft claims, verify claims, summarize verified claims. Each step's output feeds the next. The orchestrator also writes intermediate state to disk so you can inspect what each agent produced and audit any sentence in the final brief.

```python
# orchestrator.py
import json
import os
from dataclasses import asdict
from researcher import collect_sources
from drafter import draft_claims
from verifier import verify_all
from summarizer import summarize

def run_research_team(topic: str, output_dir: str = "research_output") -> str:
    os.makedirs(output_dir, exist_ok=True)

    print(f"Researching: {topic}")
    sources = collect_sources(topic)
    print(f"  Collected {len(sources)} sources")
    with open(f"{output_dir}/sources.json", "w") as f:
        json.dump([asdict(s) for s in sources], f, indent=2)

    print("Drafting claims...")
    claims = draft_claims(topic, sources)
    print(f"  Drafted {len(claims)} candidate claims")
    with open(f"{output_dir}/claims.json", "w") as f:
        json.dump([asdict(c) for c in claims], f, indent=2)

    print("Verifying claims...")
    verified = verify_all(claims, sources)
    verified_count = sum(1 for v in verified if v.status == "verified")
    print(f"  Verified {verified_count} of {len(verified)} claims")
    with open(f"{output_dir}/verified.json", "w") as f:
        json.dump([asdict(v) for v in verified], f, indent=2)

    print("Summarizing...")
    brief = summarize(topic, verified)
    with open(f"{output_dir}/brief.txt", "w") as f:
        f.write(brief)

    return brief

if __name__ == "__main__":
    topic = "How GLP-1 weight loss drugs are affecting US restaurant industry sales in 2026"
    brief = run_research_team(topic)
    print("\n=== BRIEF ===\n")
    print(brief)
```

### Step 8: Run the team and inspect the audit trail

Execute the orchestrator. Then open the four output files. Every sentence in `brief.txt` traces to a verified claim in `verified.json`, which traces to a source in `sources.json`. The audit trail is the entire point.

```bash
python orchestrator.py
ls research_output/
# sources.json - what the Researcher found
# claims.json - what the Drafter extracted
# verified.json - what the Verifier accepted, partial, or rejected
# brief.txt - the final output, every sentence traceable
```

## Breakage

Collapse the three agents into one. Give Claude the topic and a system prompt that says "research this and write a brief." Claude does all four jobs in a single call. It generates claims and supporting "evidence" in the same response, with no external source-fetching step in between. The output looks identical to the verified version. The structure is the same. The tone is the same. But there is no audit trail. Some claims trace to real sources Claude saw during training. Some claims trace to nothing at all. You cannot tell which is which. The brief reads as authoritative because that is how Claude writes. The 12% statistic is invented. The executive quote is paraphrased from a different industry. You ship the brief. The fabricated quote ends up in your investment memo.

```text
DIAGRAM: Single-agent failure mode
Caption: One generalist agent producing unverifiable output that mixes real facts with fabrications
Nodes:
1. Topic input - The research question from the user
2. Generalist agent (Claude) - Does research, drafting, verification, and summarization in one prompt
3. Output brief - Looks authoritative but contains a mix of real and invented facts
4. Reader - Cannot distinguish verified claims from hallucinations
Flow:
- Topic input goes to Generalist agent
- Generalist agent generates a brief using its training data and inference
- No external source-fetching step
- No verification step
- Output brief reaches Reader with no audit trail
- Reader trusts the brief because it reads as authoritative
- Some sentences are real, some are fabrications, no way to tell which
```

## The fix

The fix is the typed handoff between the Researcher and the Verifier from Steps 3 and 5. The Researcher must produce externally-fetched sources before any claim is drafted, and the Verifier must check each drafted claim against one of those sources. The critical guard is in the Summarizer's input filter from Step 6, isolated below for emphasis. This single line is what enforces the verification contract: the Summarizer literally cannot see unverified claims.

```python
# The constraint that makes the team trustworthy
def summarize(topic: str, verified_claims: list[VerifiedClaim]) -> str:
    # This filter is the entire trust mechanism.
    # Partial and rejected claims are removed before the Summarizer sees them.
    accepted = [c for c in verified_claims if c.status == "verified"]

    if len(accepted) < 3:
        return "Insufficient verified claims to produce a brief."

    # The Summarizer can only write about what passed verification.
    # Every sentence in the output traces back to an accepted claim,
    # which traces back to a source the Researcher fetched.
```

The pattern: the typed handoff makes it impossible for downstream agents to access upstream data they should not see. The Summarizer never receives unverified claims because the input filter strips them. There is no clever prompt engineering that can make the Summarizer fabricate, because there is no fabricatable input in its context.

## Fixed state

```text
DIAGRAM: Research team with typed handoffs enforced
Caption: Each agent receives only what its predecessor verified, with full audit trail
Nodes:
1. Topic input - The research question
2. Researcher agent - Generates queries, fetches sources from external API
3. External Search API - Returns real source URLs and excerpts
4. Sources collection (typed) - {url, title, excerpt, fetched_at}
5. Drafter - Extracts candidate claims, each citing a source URL
6. Claims collection (typed) - {text, suggested_source_url}
7. Verifier agent - Checks each claim against its cited source
8. Verified claims collection (typed) - {text, status, source_url, evidence}
9. Filter - Strips claims where status is not "verified"
10. Summarizer agent - Writes brief using only verified claims
11. Final brief - Every sentence traces to a verified claim, which traces to a fetched source
12. Audit files - sources.json, claims.json, verified.json, brief.txt for human inspection
Flow:
- Topic input reaches Researcher agent
- Researcher agent queries External Search API for each generated query
- External Search API returns results, Researcher assembles Sources collection
- Sources collection goes to Drafter
- Drafter produces Claims collection with source citations
- Claims collection plus Sources collection reach Verifier agent
- Verifier agent checks each claim against its cited source, produces Verified claims collection
- Filter removes any claim where status is not "verified"
- Summarizer agent receives only verified claims, writes Final brief
- All intermediate collections write to Audit files for human inspection
```

## After

You ask the research team about GLP-1 drugs and the restaurant industry. Five minutes later you have a 300-word brief. Every sentence has a citation marker. You open `verified.json` and see 11 claims marked verified, 4 marked partial, 6 marked rejected. The 12% dessert statistic from your old failed attempt? It appears in claims.json, but verified.json shows it was rejected with the evidence "source mentions declining sales but provides no specific percentage." It never reached the brief. The executive quote? Verified, with the exact source URL and the matching sentence from the original article. You send the brief to your investment partner with the audit files attached. Your partner reads three citations at random, finds them all accurate, and trusts the rest. You research three more topics that afternoon. None of the briefs contain a fabrication. The team is faster than you, more thorough than you, and more honest than a generalist agent because each member only has one job and is constrained by the typed contract from the previous member.

## Takeaway

The pattern is: separate concerns into specialized agents and force them to communicate through typed contracts. Each agent gets a narrow job description and a strict input/output schema. Trust comes not from prompt engineering but from the structure of the handoffs. A downstream agent cannot use information that the upstream agent did not produce in the correct shape. Apply this anywhere a single LLM call has too many responsibilities: research, code review, content moderation, customer support routing. Specialization plus typed handoffs beats generalist agents every time, because the structure does the constraint work that prompts cannot reliably do.
