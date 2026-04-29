## Opening thesis

You will build three role-specific agents, each with a scoped system prompt, each calling the Anthropic API independently. An agent with a well-scoped role-specific system prompt produces deterministically specialized output on every call, where a human generalist switching between roles loses context and degrades over the day. The composition of researcher, writer, and editor outputs is clean because the roles never bleed into each other.

## Before

You have one generic agent handling researcher, writer, and editor roles, and the output reads like all three are fighting for the keyboard. The researcher inserts half-formed opinions. The writer drops in raw URLs mid-paragraph. The editor rewrites entire sections instead of flagging issues. You prompt it to "research this topic, then write a draft, then edit it," and the result is a 900-word blob where citations live next to style critiques next to creative metaphors. Every time you adjust the prompt to improve research quality, the writing voice shifts. Every time you tighten the editing instructions, the research section gets thinner. One system prompt is doing three jobs. It does none of them well.

## Architecture

The system runs three separate API calls in sequence. Each call sends a different system prompt and receives a different type of output. The researcher produces structured facts. The writer consumes those facts and produces prose. The editor consumes the prose and produces a final version with tracked changes. No agent sees another agent's system prompt.

```text
DIAGRAM: Three-agent pipeline
Caption: Sequential calls to the Anthropic API, each with a role-specific system prompt
Nodes:
1. Orchestrator (Python script) - sends requests, passes outputs between stages
2. Researcher Agent (API call 1) - returns structured research notes
3. Writer Agent (API call 2) - returns a prose draft from research notes
4. Editor Agent (API call 3) - returns a final draft with change annotations
Flow:
- Orchestrator sends topic to Researcher Agent with researcher system prompt
- Researcher Agent returns JSON research notes to Orchestrator
- Orchestrator sends research notes to Writer Agent with writer system prompt
- Writer Agent returns prose draft to Orchestrator
- Orchestrator sends prose draft to Editor Agent with editor system prompt
- Editor Agent returns edited final draft to Orchestrator
```

## Step-by-step implementation

### Step 1: Set up the environment

Install the Anthropic Python SDK and set your API key. Get your key from https://console.anthropic.com/settings/keys. Export it as an environment variable.

```bash
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-...your-key-here"
```

### Step 2: Define the researcher system prompt

This prompt constrains the agent to research behavior only. It must return structured JSON. It must not write prose, offer opinions, or suggest edits. The output format is locked so the writer agent can parse it.

```python
# prompts.py

RESEARCHER_SYSTEM_PROMPT = """You are a research specialist. Your only job is to gather and organize factual information about a given topic.

Rules:
1. Return ONLY valid JSON. No prose before or after the JSON.
2. Never write narrative paragraphs. Never offer stylistic suggestions.
3. Never edit or critique. You are not a writer or an editor.
4. Structure your output as:
{
  "topic": "<the topic>",
  "key_facts": ["fact 1", "fact 2", ...],
  "sources": ["source description 1", "source description 2", ...],
  "open_questions": ["question 1", "question 2", ...]
}
5. Include 5 to 10 key facts. Include 2 to 5 source descriptions. Include 1 to 3 open questions.
6. Each fact must be a single declarative sentence.
7. If you are unsure about a fact, omit it. Do not hedge with words like "possibly" or "arguably".
"""
```

### Step 3: Define the writer system prompt

This prompt constrains the agent to writing behavior only. It receives research notes as input. It must not add new facts, remove facts, or comment on the quality of the research.

```python
# prompts.py (continued)

WRITER_SYSTEM_PROMPT = """You are a prose writer. Your only job is to turn structured research notes into a clear, readable article draft.

Rules:
1. Write 300 to 500 words of prose.
2. Use every key fact from the research notes. Do not invent new facts.
3. Do not include raw JSON in your output. Transform the facts into natural sentences.
4. Do not critique the research. Do not flag missing information.
5. Do not edit your own work. No meta-commentary like "this section could be stronger."
6. Write in a neutral, informative tone. Short sentences. Active voice.
7. Include a one-sentence introduction and a one-sentence conclusion.
8. Reference the sources naturally within the text where relevant.
"""
```

### Step 4: Define the editor system prompt

This prompt constrains the agent to editing behavior only. It receives a prose draft. It must not do new research or rewrite sections wholesale.

```python
# prompts.py (continued)

EDITOR_SYSTEM_PROMPT = """You are a copy editor. Your only job is to improve clarity, grammar, and flow of a given draft.

Rules:
1. Return the full edited text.
2. After the edited text, add a section titled "CHANGES:" listing each change you made and why.
3. Do not add new facts or claims. If a fact seems wrong, flag it in CHANGES but do not remove it.
4. Do not conduct research. Do not add source references that were not in the original.
5. Do not rewrite more than one sentence at a time. Preserve the author's voice.
6. Fix grammar errors, awkward phrasing, and unclear transitions.
7. If the draft is already clean, return it unchanged and state "No changes needed" in the CHANGES section.
"""
```

### Step 5: Build the orchestrator

The orchestrator calls each agent in sequence. It passes the output of one agent as the user message to the next. Each call uses a different system prompt.

```python
# orchestrator.py

import anthropic
import json
from prompts import RESEARCHER_SYSTEM_PROMPT, WRITER_SYSTEM_PROMPT, EDITOR_SYSTEM_PROMPT

client = anthropic.Anthropic()
MODEL = "claude-sonnet-4-20250514"

def call_agent(system_prompt: str, user_message: str) -> str:
    response = client.messages.create(
        model=MODEL,
        max_tokens=2048,
        system=system_prompt,
        messages=[{"role": "user", "content": user_message}]
    )
    return response.content[0].text

def run_pipeline(topic: str) -> dict:
    print("Stage 1: Research")
    research_raw = call_agent(
        RESEARCHER_SYSTEM_PROMPT,
        f"Research this topic: {topic}"
    )
    print(research_raw)

    print("\nStage 2: Write")
    draft = call_agent(
        WRITER_SYSTEM_PROMPT,
        f"Write an article from these research notes:\n\n{research_raw}"
    )
    print(draft)

    print("\nStage 3: Edit")
    final = call_agent(
        EDITOR_SYSTEM_PROMPT,
        f"Edit this draft:\n\n{draft}"
    )
    print(final)

    return {
        "research": research_raw,
        "draft": draft,
        "final": final
    }

if __name__ == "__main__":
    result = run_pipeline("The history and impact of container shipping on global trade")
```

### Step 6: Run the pipeline

Execute the orchestrator. You should see three distinct output blocks. The research block is JSON. The draft block is prose. The edit block is prose with a CHANGES section.

```bash
python orchestrator.py
```

### Step 7: Verify role isolation

This is the accountability step. You need to confirm that each agent stayed in its lane. This script parses the research output as JSON, checks the draft for JSON fragments, and checks the editor output for new factual claims.

```python
# verify.py

import json
import sys
from orchestrator import run_pipeline

def verify_pipeline(topic: str) -> bool:
    result = run_pipeline(topic)
    passed = True

    # Check 1: Research output must be valid JSON
    try:
        research_data = json.loads(result["research"])
        assert "key_facts" in research_data, "Missing key_facts in research"
        assert "sources" in research_data, "Missing sources in research"
        print("PASS: Research output is valid structured JSON.")
    except (json.JSONDecodeError, AssertionError) as e:
        print(f"FAIL: Research output is not valid JSON. Error: {e}")
        passed = False

    # Check 2: Draft must not contain JSON braces (role bleed)
    if result["draft"].strip().startswith("{"):
        print("FAIL: Draft starts with JSON. Writer is acting like researcher.")
        passed = False
    else:
        print("PASS: Draft is prose, not JSON.")

    # Check 3: Editor must include CHANGES section
    if "CHANGES:" in result["final"]:
        print("PASS: Editor included change annotations.")
    else:
        print("FAIL: Editor did not include CHANGES section.")
        passed = False

    # Check 4: Editor must not contain research patterns
    research_signals = ['"key_facts"', '"sources"', '"open_questions"']
    for signal in research_signals:
        if signal in result["final"]:
            print(f"FAIL: Editor output contains research pattern: {signal}")
            passed = False

    return passed

if __name__ == "__main__":
    topic = "The history and impact of container shipping on global trade"
    ok = verify_pipeline(topic)
    sys.exit(0 if ok else 1)
```

### Step 8: Run verification

Run the verification script. All checks should pass. If any fail, the system prompt for the offending agent needs tighter constraints.

```bash
python verify.py
```

## Breakage

If you skip the verification step, role bleed goes undetected. The researcher starts writing prose paragraphs instead of returning JSON. The writer injects hedges like "more research is needed." The editor rewrites three paragraphs and adds claims that came from nowhere. You ship the output thinking it works. Two weeks later you change the writer prompt and suddenly the editor output breaks, because the writer had been compensating for a weak editor prompt all along. Without verification, you cannot tell which agent drifted.

```text
DIAGRAM: Unverified pipeline with role bleed
Caption: Without verification, role drift compounds through the pipeline
Nodes:
1. Orchestrator - passes outputs between stages, no checks
2. Researcher Agent - returns a mix of JSON and prose opinions
3. Writer Agent - receives malformed input, compensates by researching
4. Editor Agent - receives a draft full of raw facts, rewrites everything
Flow:
- Orchestrator sends topic to Researcher Agent
- Researcher Agent returns half-JSON, half-prose to Orchestrator
- Orchestrator passes broken output to Writer Agent
- Writer Agent guesses at structure, adds its own facts
- Orchestrator passes bloated draft to Editor Agent
- Editor Agent rewrites entire draft, removing original research
```

## The fix

Add a validation gate between each stage. If the research output fails JSON parsing, retry the researcher call once. If the draft contains JSON fragments, retry the writer call. If the editor output lacks a CHANGES section, retry the editor call. This turns silent drift into a hard failure that you see immediately.

```python
# orchestrator.py (updated run_pipeline function)

import json

def validate_research(raw: str) -> dict:
    data = json.loads(raw)
    assert "key_facts" in data, "Research missing key_facts"
    assert "sources" in data, "Research missing sources"
    return data

def validate_draft(text: str) -> str:
    assert not text.strip().startswith("{"), "Draft is JSON, not prose"
    assert len(text.split()) >= 100, "Draft is too short"
    return text

def validate_edit(text: str) -> str:
    assert "CHANGES:" in text, "Editor did not include CHANGES section"
    return text

def call_with_retry(system_prompt: str, user_message: str, validator, retries: int = 1):
    for attempt in range(retries + 1):
        raw = call_agent(system_prompt, user_message)
        try:
            return validator(raw), raw
        except (json.JSONDecodeError, AssertionError) as e:
            if attempt == retries:
                raise RuntimeError(f"Agent failed validation after {retries + 1} attempts: {e}")
            print(f"Retry {attempt + 1}: {e}")

def run_pipeline(topic: str) -> dict:
    print("Stage 1: Research")
    research_data, research_raw = call_with_retry(
        RESEARCHER_SYSTEM_PROMPT,
        f"Research this topic: {topic}",
        validate_research
    )
    print(research_raw)

    print("\nStage 2: Write")
    _, draft = call_with_retry(
        WRITER_SYSTEM_PROMPT,
        f"Write an article from these research notes:\n\n{research_raw}",
        validate_draft
    )
    print(draft)

    print("\nStage 3: Edit")
    _, final = call_with_retry(
        EDITOR_SYSTEM_PROMPT,
        f"Edit this draft:\n\n{draft}",
        validate_edit
    )
    print(final)

    return {"research": research_raw, "draft": draft, "final": final}
```

## Fixed state

```text
DIAGRAM: Validated three-agent pipeline
Caption: Each stage validates output before passing to the next agent
Nodes:
1. Orchestrator - sends requests, validates outputs, retries on failure
2. Researcher Agent - returns structured JSON research notes
3. Research Validator - confirms valid JSON with required fields
4. Writer Agent - returns prose draft from research notes
5. Draft Validator - confirms output is prose, not JSON, meets length
6. Editor Agent - returns edited draft with change annotations
7. Edit Validator - confirms CHANGES section exists
Flow:
- Orchestrator sends topic to Researcher Agent
- Researcher Agent returns JSON to Research Validator
- Research Validator passes or triggers retry
- Orchestrator sends validated research to Writer Agent
- Writer Agent returns prose to Draft Validator
- Draft Validator passes or triggers retry
- Orchestrator sends validated draft to Editor Agent
- Editor Agent returns edited text to Edit Validator
- Edit Validator passes or triggers retry
- Orchestrator collects final validated output
```

## After

Three agents, three system prompts, three distinct outputs. The researcher returns valid JSON every time. The writer produces prose that uses every fact from the research and adds nothing. The editor makes small corrections and lists what it changed. You modify the writer prompt to adjust tone. The research output does not change. The editor output adapts to the new tone without breaking its own rules. Each prompt is a single point of control for a single behavior. The composition is clean because the roles never share a context window.

## Takeaway

The pattern is constraint through isolation. One system prompt per role. One validation gate per boundary. When you need a new behavior, you add a new agent with a new prompt. You do not widen an existing prompt. Narrow prompts produce predictable agents. Predictable agents compose into reliable systems.