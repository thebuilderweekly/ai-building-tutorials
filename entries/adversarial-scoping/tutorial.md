## Opening thesis

You will build a Python script that kills a bad product idea before you spend a week building it. The script feeds a PRD through six adversarial critique passes using the Anthropic API, each pass wearing a different hostile hat. An agent runs six adversarial critique passes in two minutes and surfaces objections a solo founder never asks themselves, where a human review panel takes three days to assemble and half the panel will be polite. The output is a ranked list of the three objections most likely to kill the product idea before any code gets written.

## Before

You finished your PRD on Sunday night. It felt sharp. Monday morning you started building: database schema, auth flow, the first API route. By Wednesday you had 40 files. Thursday you showed it to a friend who builds in the same space. She said one sentence: "Your user doesn't have that problem." You sat with it. She was right. Four days of code went into the trash. The worst part is you knew the question existed. You just never asked it, because you were the author and the reviewer at the same time. Nobody plays devil's advocate on their own work. Not honestly. Not at 1 a.m. when the idea feels alive. You needed six skeptics in a room. You had zero.

## Architecture

The system is a single Python script that reads a PRD from a local file, sends it through six sequential API calls to Claude, and writes a final verdict file. Each call uses a different adversarial persona encoded in the system prompt. A final synthesis call ranks the objections by severity.

```text
DIAGRAM: Adversarial PRD Review Pipeline
Caption: Six critique passes feed into one synthesis pass, producing a ranked kill list.
Nodes:
1. prd.md - The raw PRD file on disk
2. critique_runner.py - Python script that orchestrates all API calls
3. Persona 1: Customer Skeptic - Challenges whether the user actually has the stated problem
4. Persona 2: Market Cynic - Challenges whether the market is real and reachable
5. Persona 3: Technical Saboteur - Finds architectural risks the author glossed over
6. Persona 4: Unit Economics Auditor - Asks if the business math works at scale
7. Persona 5: Competitor Analyst - Identifies existing solutions the author ignored
8. Persona 6: Scope Creep Detector - Finds hidden complexity that doubles the timeline
9. Claude API (Anthropic) - The model endpoint processing each persona call
10. synthesis call - One final call that ranks all objections
11. verdict.md - The output file with ranked objections
Flow:
- prd.md is read by critique_runner.py
- critique_runner.py sends prd.md content to Claude API six times, once per persona
- Each persona response is collected into a list
- critique_runner.py sends all six responses to Claude API as a synthesis call
- Synthesis response is written to verdict.md
```

## Step-by-step implementation

### Step 1: Set up the project and install the Anthropic SDK

Create a directory and install the one dependency. The Anthropic Python SDK wraps the Messages API. Get your API key from https://console.anthropic.com/settings/keys and export it.

```bash
mkdir adversarial-review && cd adversarial-review
python3 -m venv venv
source venv/bin/activate
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-...your-key-here"
```

### Step 2: Create a sample PRD to test against

You need a real PRD to feed the system. This sample describes a plausible but flawed idea: an AI meal planner for college students. Save it as `prd.md`. Replace it with your own PRD when you run this for real.

```bash
cat > prd.md << 'EOF'
# Product Requirements Document: MealMind

**Problem.** College students eat poorly because they lack time and knowledge to plan meals around a tight budget.

**Solution.** A generative mobile app that produces weekly meal plans optimized for a student's budget, dietary restrictions, and dorm kitchen constraints.

**Target user.** US college students aged 18 to 22 living in dorms or off-campus apartments.

**Core features.**
1. Budget input: user sets weekly grocery budget (default $40).
2. Generated meal plan: 7 days, 3 meals per day.
3. Grocery list export: one-tap list synced to Instacart or Walmart.
4. Dietary filters: vegan, gluten-free, halal, nut-free.
5. Leftover tracking: mark unused ingredients, get next-day recipes.

**Revenue model.** Freemium. Free tier: 2 meal plans per month. Pro tier: $4.99/month for unlimited plans and grocery sync.

**Timeline.** MVP in 6 weeks. Solo founder. React Native frontend, Python backend, OpenAI API for generation.

**Success metric.** 1,000 paying subscribers within 6 months of launch.
EOF
```

### Step 3: Define the six adversarial personas

Each persona is a system prompt that tells Claude to adopt a specific hostile stance. The key design choice: each persona must end with "List your top 3 objections, each with a severity rating of LOW, MEDIUM, or HIGH." This forces structured output.

```python
# personas.py

PERSONAS = [
    {
        "name": "Customer Skeptic",
        "system": (
            "You are a ruthless customer researcher. Your job is to find evidence that the target user "
            "does not actually have the stated problem, or that they have it but will never pay to solve it. "
            "Assume nothing. Challenge every claim about user behavior. "
            "List your top 3 objections, each with a severity rating of LOW, MEDIUM, or HIGH."
        ),
    },
    {
        "name": "Market Cynic",
        "system": (
            "You are a market analyst who has seen 500 failed startups. Your job is to find reasons this "
            "market is fake, too small, or unreachable. Challenge the TAM, the distribution channel, and "
            "the timing. Be specific. "
            "List your top 3 objections, each with a severity rating of LOW, MEDIUM, or HIGH."
        ),
    },
    {
        "name": "Technical Saboteur",
        "system": (
            "You are a senior engineer who reviews architecture documents for hidden risk. Your job is to "
            "find the technical assumptions that will blow up during implementation. Focus on integration "
            "complexity, data dependencies, and scaling bottlenecks. "
            "List your top 3 objections, each with a severity rating of LOW, MEDIUM, or HIGH."
        ),
    },
    {
        "name": "Unit Economics Auditor",
        "system": (
            "You are a finance person who kills products that cannot make money. Your job is to calculate "
            "whether the revenue model covers the cost of acquisition, infrastructure, and API calls. "
            "Use rough estimates. Show your math. "
            "List your top 3 objections, each with a severity rating of LOW, MEDIUM, or HIGH."
        ),
    },
    {
        "name": "Competitor Analyst",
        "system": (
            "You are a competitive intelligence analyst. Your job is to name existing products, features, "
            "or free alternatives that already solve this problem. If the PRD does not mention competitors, "
            "that is itself a red flag. Be specific with product names and URLs where possible. "
            "List your top 3 objections, each with a severity rating of LOW, MEDIUM, or HIGH."
        ),
    },
    {
        "name": "Scope Creep Detector",
        "system": (
            "You are a project manager who has shipped 50 products. Your job is to find hidden complexity "
            "in the feature list that will double the stated timeline. Look for features that sound like "
            "one task but are actually five. Challenge the MVP definition. "
            "List your top 3 objections, each with a severity rating of LOW, MEDIUM, or HIGH."
        ),
    },
]
```

### Step 4: Build the critique runner

This script reads the PRD, loops through all six personas, collects responses, and stores them for synthesis. Each call uses `claude-sonnet-4-20250514` with a max of 1024 tokens. That keeps cost under $0.15 for the full run.

```python
# critique_runner.py

import os
import json
import anthropic
from personas import PERSONAS

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env

def load_prd(path: str) -> str:
    with open(path, "r") as f:
        return f.read()

def run_critique(prd_text: str, persona: dict) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system=persona["system"],
        messages=[
            {
                "role": "user",
                "content": f"Here is a Product Requirements Document. Critique it.\n\n{prd_text}",
            }
        ],
    )
    text = response.content[0].text
    return {"persona": persona["name"], "critique": text}

def run_all_critiques(prd_path: str) -> list:
    prd_text = load_prd(prd_path)
    results = []
    for persona in PERSONAS:
        print(f"Running critique: {persona['name']}...")
        result = run_critique(prd_text, persona)
        results.append(result)
    return results

if __name__ == "__main__":
    critiques = run_all_critiques("prd.md")
    with open("critiques.json", "w") as f:
        json.dump(critiques, f, indent=2)
    print(f"Saved {len(critiques)} critiques to critiques.json")
```

### Step 5: Run the six critiques

Execute the runner. On a typical PRD this takes 30 to 90 seconds depending on response length. You will see each persona name print as it completes.

```bash
python critique_runner.py
```

### Step 6: Build the synthesis step

The synthesis call takes all six critiques and asks Claude to rank every objection by severity, then pick the top three. This is the step that turns raw noise into a decision.

```python
# synthesize.py

import json
import anthropic

client = anthropic.Anthropic()

def load_critiques(path: str) -> list:
    with open(path, "r") as f:
        return json.load(f)

def synthesize(critiques: list) -> str:
    combined = ""
    for c in critiques:
        combined += f"### {c['persona']}\n{c['critique']}\n\n"

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system=(
            "You are a product decision advisor. You have received six adversarial critiques of a PRD. "
            "Your job: read all 18 objections, deduplicate them, rank them by severity, and output "
            "the top 3 objections that are most likely to kill the product. For each objection, state "
            "the persona that raised it, the objection in one sentence, the severity (HIGH/MEDIUM/LOW), "
            "and one concrete action the founder should take before writing any code. "
            "Format the output as markdown with numbered items."
        ),
        messages=[
            {
                "role": "user",
                "content": f"Here are the six adversarial critiques:\n\n{combined}",
            }
        ],
    )
    return response.content[0].text

if __name__ == "__main__":
    critiques = load_critiques("critiques.json")
    verdict = synthesize(critiques)
    with open("verdict.md", "w") as f:
        f.write("# Adversarial Review Verdict\n\n")
        f.write(verdict)
    print("Verdict written to verdict.md")
```

### Step 7: Run the synthesis

This produces the final verdict file. Open it and read the three objections. If any of them make you uncomfortable, good. That discomfort on day zero is worth more than the same realization on day four.

```bash
python synthesize.py
cat verdict.md
```

### Step 8: Wire both steps into a single command

A one-line runner makes this something you actually use instead of something you bookmark and forget.

```bash
cat > review.sh << 'SCRIPT'
#!/usr/bin/env bash
set -e
echo "Starting adversarial review..."
python critique_runner.py
python synthesize.py
echo "Done. Read verdict.md."
SCRIPT
chmod +x review.sh
./review.sh
```

## Breakage

If you skip the synthesis step, you get 18 objections with no ranking. A solo founder will read all 18, feel overwhelmed, and ignore every one of them. Raw critiques without triage are noise. Noise feels like due diligence but produces no decisions. The founder goes back to building the thing they already wanted to build, now with a vague sense of guilt instead of a concrete action list. The six personas did their jobs. The pipeline failed at the last mile.

```text
DIAGRAM: Failure Without Synthesis
Caption: Six critiques produce 18 unranked objections that the founder ignores.
Nodes:
1. prd.md - The raw PRD
2. critique_runner.py - Sends PRD through six personas
3. critiques.json - 18 unranked objections
4. Founder's brain - Overwhelmed, defaults to original plan
Flow:
- prd.md feeds into critique_runner.py
- critique_runner.py produces critiques.json with 18 objections
- Founder reads critiques.json, gets overwhelmed
- Founder ignores objections and starts building anyway
```

## The fix

The synthesis step is the fix. It already exists in Step 6 above, but here is the hardened version that also rejects a PRD outright if two or more HIGH severity objections survive synthesis. This version adds a go/no-go signal so the founder does not have to interpret the output.

```python
# In synthesize.py, replace the __main__ block with this:

if __name__ == "__main__":
    critiques = load_critiques("critiques.json")
    verdict = synthesize(critiques)

    high_count = verdict.upper().count("HIGH")
    if high_count >= 2:
        signal = "NO-GO: Two or more HIGH severity objections found. Do not build until resolved."
    elif high_count == 1:
        signal = "CAUTION: One HIGH severity objection found. Resolve it before writing code."
    else:
        signal = "GO: No HIGH severity objections. Proceed with awareness of MEDIUM risks."

    with open("verdict.md", "w") as f:
        f.write("# Adversarial Review Verdict\n\n")
        f.write(f"## Signal: {signal}\n\n")
        f.write(verdict)

    print(f"\n{signal}")
    print("Full verdict written to verdict.md")
```

## Fixed state

```text
DIAGRAM: Complete Pipeline With Go/No-Go Signal
Caption: Six critiques feed into synthesis, which produces a ranked verdict with a binary decision signal.
Nodes:
1. prd.md - The raw PRD
2. critique_runner.py - Sends PRD through six personas
3. Claude API - Processes each persona call
4. critiques.json - Six structured critique responses
5. synthesize.py - Deduplicates, ranks, and counts HIGH objections
6. verdict.md - Top 3 objections plus GO, CAUTION, or NO-GO signal
Flow:
- prd.md feeds into critique_runner.py
- critique_runner.py makes six calls to Claude API
- Claude API returns six critiques stored in critiques.json
- synthesize.py reads critiques.json and makes one synthesis call to Claude API
- synthesize.py counts HIGH severity objections and assigns a signal
- Output written to verdict.md with signal and ranked objections
```

## After

You finished your PRD on Sunday night. It felt sharp. Monday morning you ran `./review.sh`. Two minutes later you opened `verdict.md`. The signal said NO-GO. The top objection: "Your target user has a $40/week grocery budget but you are charging $4.99/month for a tool that competes with free recipe apps and TikTok meal prep videos. Customer acquisition cost will exceed lifetime value." The second objection: "Instacart and Walmart integrations each require partner agreements with 4 to 8 week approval cycles, making your 6-week MVP timeline impossible." You sat with it. The objections were right. You rewrote the PRD before lunch. You did not throw away four days of code because there was no code to throw away.

## Takeaway

The pattern is adversarial decomposition: split one vague review task into multiple hostile perspectives, run them in parallel, and force a synthesis step that produces a decision, not a discussion. This works for PRDs, architecture docs, pitch decks, and pricing pages. Anything you wrote and plan to act on deserves six enemies before it gets a single friend.
