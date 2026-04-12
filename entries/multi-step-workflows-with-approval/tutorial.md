## Opening thesis

You will build a state-machine agent that executes a multi-step content-publishing workflow, pausing for human approval before any step that spends money or touches a public audience. An autonomous agent doing a 10-step task fully autonomously will eventually take a wrong path costing real money or trust; a state-machine agent with approval gates at the right steps gets the same speed with bounded risk. The agent uses Anthropic's Claude API for reasoning and Cue API for orchestrating the state machine with built-in human-in-the-loop checkpoints.

## Before

You have a 10-step agent workflow. It researches a topic, drafts copy, picks images, sets a budget, configures ad targeting, and publishes. Steps 1 through 6 run fine. Step 7 picks a $4,000 daily ad budget instead of $400 because the LLM misread a constraint. Steps 8 through 10 execute without hesitation. You discover the mistake 14 hours later when your billing alert fires. The agent did exactly what you told it to do: run autonomously. The problem is that "autonomously" included the part where it burned your quarterly ad budget in a single afternoon. Every team that has shipped an agent end-to-end has a story like this. The dollar amounts vary. The embarrassment does not.

## Architecture

The system is a finite state machine with 10 states. Each state maps to one step of the publishing workflow. Three of those states are "gated": the machine transitions into a PENDING_APPROVAL status and sends a notification through Cue API. The workflow resumes only after a human approves or corrects the proposed action. Claude handles the reasoning at each step. Cue API holds the state, enforces the gates, and delivers the approval request.

```text
DIAGRAM: Multi-step publishing workflow with approval gates
Caption: A 10-state machine where states 4, 7, and 9 require human approval before the agent proceeds.
Nodes:
1. Claude API: Generates decisions at each workflow step (draft text, pick budget, select audience, etc.)
2. Cue API: Manages state machine, persists workflow state, enforces approval gates, sends notifications
3. Human reviewer: Receives approval requests at gated steps, approves or corrects
4. State store: Cue API's internal persistence layer holding current state and step outputs
Flow:
- Claude API produces a proposed action for the current step
- Cue API records the action and checks if the current state is gated
- If not gated: Cue API auto-advances to the next state
- If gated: Cue API sets status to PENDING_APPROVAL and notifies the human reviewer
- Human reviewer approves or sends a correction through Cue API
- Cue API advances the state machine to the next state with the approved (or corrected) action
- Loop repeats until the final state completes
```

## Step-by-step implementation

### 1. Set up environment variables

Both APIs require keys. Get your Anthropic key from https://console.anthropic.com/settings/keys. Get your Cue API key from https://app.cueapi.com/settings/api-keys. Store them as environment variables so no secrets appear in code.

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
export CUEAPI_API_KEY="cue_..."
```

### 2. Install dependencies

The project uses Python. You need the Anthropic SDK and the requests library for Cue API calls.

```bash
pip install anthropic requests
```

### 3. Define the state machine

Create a file called `workflow.py`. Define the 10 steps and mark which ones are gated. The `gated` flag is what separates a runaway agent from a bounded one.

```python
# workflow.py

STEPS = [
    {"name": "research_topic", "gated": False, "prompt": "Research the topic '{topic}' and return 5 key points as JSON."},
    {"name": "draft_headline", "gated": False, "prompt": "Write 3 headline options for an article about: {research_topic}"},
    {"name": "draft_body", "gated": False, "prompt": "Write a 200-word article body using this headline: {draft_headline}"},
    {"name": "select_image_keywords", "gated": True, "prompt": "Suggest 3 stock photo search queries for this article: {draft_body}"},
    {"name": "set_ad_budget", "gated": True, "prompt": "Given a monthly budget of $1200 and a 30-day campaign, propose a daily ad budget in USD. Return only the number."},
    {"name": "define_audience", "gated": False, "prompt": "Define a target audience for this article: {draft_body}"},
    {"name": "configure_ad_targeting", "gated": True, "prompt": "Based on audience '{define_audience}', propose ad targeting parameters as JSON with keys: age_min, age_max, interests, locations."},
    {"name": "generate_social_post", "gated": False, "prompt": "Write a social media post promoting this article: {draft_headline}"},
    {"name": "schedule_publication", "gated": True, "prompt": "Propose a publication date and time (ISO 8601) for this article, considering it is currently April 2026."},
    {"name": "final_summary", "gated": False, "prompt": "Summarize the full campaign plan given all previous steps."}
]
```

### 4. Create the workflow in Cue API

Register the workflow with Cue API. This call creates a persistent workflow instance that survives restarts. The approval gates are stored server-side so the agent cannot skip them even if the local process crashes and restarts.

```python
# create_workflow.py
import os
import requests
from workflow import STEPS

CUEAPI_BASE = "https://api.cueapi.com/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['CUEAPI_API_KEY']}",
    "Content-Type": "application/json"
}

def create_workflow(topic: str) -> str:
    steps_payload = []
    for i, step in enumerate(STEPS):
        steps_payload.append({
            "name": step["name"],
            "order": i,
            "requires_approval": step["gated"]
        })

    body = {
        "name": f"publish-campaign-{topic.replace(' ', '-')[:30]}",
        "steps": steps_payload,
        "metadata": {"topic": topic}
    }

    resp = requests.post(f"{CUEAPI_BASE}/workflows", json=body, headers=HEADERS)
    resp.raise_for_status()
    workflow_id = resp.json()["id"]
    print(f"Created workflow: {workflow_id}")
    return workflow_id

if __name__ == "__main__":
    wf_id = create_workflow("zero-downtime database migrations")
    print(wf_id)
```

### 5. Build the agent loop

This is the core. The agent iterates through each step. For non-gated steps, it calls Claude, records the result, and moves on. For gated steps, it calls Claude, records the proposed action, and then waits for human approval before advancing.

```python
# agent.py
import os
import time
import requests
import anthropic
from workflow import STEPS

CUEAPI_BASE = "https://api.cueapi.com/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['CUEAPI_API_KEY']}",
    "Content-Type": "application/json"
}

client = anthropic.Anthropic()

def call_claude(prompt: str) -> str:
    message = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text

def resolve_prompt(step_index: int, results: dict, topic: str) -> str:
    template = STEPS[step_index]["prompt"]
    template = template.replace("{topic}", topic)
    for key, value in results.items():
        template = template.replace("{" + key + "}", str(value)[:500])
    return template

def run_step(workflow_id: str, step_index: int, results: dict, topic: str) -> str:
    step = STEPS[step_index]
    prompt = resolve_prompt(step_index, results, topic)
    proposed_action = call_claude(prompt)

    # Record the proposed action in Cue API
    requests.post(
        f"{CUEAPI_BASE}/workflows/{workflow_id}/steps/{step['name']}/propose",
        json={"proposed_action": proposed_action},
        headers=HEADERS
    ).raise_for_status()

    if step["gated"]:
        print(f"\n[GATED] Step '{step['name']}' needs approval.")
        print(f"Proposed action:\n{proposed_action}\n")
        return wait_for_approval(workflow_id, step["name"], proposed_action)
    else:
        # Auto-advance
        requests.post(
            f"{CUEAPI_BASE}/workflows/{workflow_id}/steps/{step['name']}/advance",
            json={"final_action": proposed_action},
            headers=HEADERS
        ).raise_for_status()
        return proposed_action

def wait_for_approval(workflow_id: str, step_name: str, proposed: str) -> str:
    print(f"Waiting for approval on step '{step_name}'...")
    while True:
        resp = requests.get(
            f"{CUEAPI_BASE}/workflows/{workflow_id}/steps/{step_name}/status",
            headers=HEADERS
        )
        resp.raise_for_status()
        data = resp.json()
        if data["status"] == "approved":
            final = data.get("corrected_action", proposed)
            print(f"Step '{step_name}' approved.")
            return final
        if data["status"] == "rejected":
            print(f"Step '{step_name}' rejected. Workflow halted.")
            raise SystemExit(1)
        time.sleep(5)

def run_workflow(workflow_id: str, topic: str):
    results = {}
    for i, step in enumerate(STEPS):
        print(f"\n--- Step {i+1}/{len(STEPS)}: {step['name']} ---")
        result = run_step(workflow_id, i, results, topic)
        results[step["name"]] = result
        print(f"Result: {result[:200]}..." if len(result) > 200 else f"Result: {result}")
    print("\nWorkflow complete.")
    return results
```

### 6. Run the agent

Combine creation and execution. The agent starts, runs the non-gated steps at full speed, and pauses at each gate.

```python
# main.py
from create_workflow import create_workflow
from agent import run_workflow

topic = "zero-downtime database migrations"
workflow_id = create_workflow(topic)
results = run_workflow(workflow_id, topic)
```

```bash
python main.py
```

### 7. Approve or correct gated steps

When the agent pauses, you review the proposed action. Use a simple curl command (or the Cue API dashboard) to approve or send a correction. This is the moment that prevents a $4,000 mistake.

```bash
# Approve as-is
curl -X POST "https://api.cueapi.com/v1/workflows/{WORKFLOW_ID}/steps/set_ad_budget/approve" \
  -H "Authorization: Bearer $CUEAPI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"approved": true}'

# Approve with correction
curl -X POST "https://api.cueapi.com/v1/workflows/{WORKFLOW_ID}/steps/set_ad_budget/approve" \
  -H "Authorization: Bearer $CUEAPI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"approved": true, "corrected_action": "40"}'
```

### 8. Add a webhook for real-time notifications

Polling works. Webhooks work better. Register a webhook so Cue API pushes approval requests to Slack, email, or your own endpoint the moment a gated step fires.

```python
# register_webhook.py
import os
import requests

CUEAPI_BASE = "https://api.cueapi.com/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['CUEAPI_API_KEY']}",
    "Content-Type": "application/json"
}

resp = requests.post(
    f"{CUEAPI_BASE}/webhooks",
    json={
        "url": "https://your-server.com/webhooks/cue-approval",
        "events": ["step.pending_approval"]
    },
    headers=HEADERS
)
resp.raise_for_status()
print(f"Webhook registered: {resp.json()['id']}")
```

## Breakage

Remove the gated steps. Set every step's `gated` field to `False`. The agent runs all 10 steps in about 45 seconds. Claude proposes a daily ad budget of $400. Except this time, it proposes $4,000 because the prompt said "monthly budget of $1200" and Claude divided by 0.3 instead of 30. The agent records $4,000, configures targeting for that budget, schedules publication, and finishes. Nobody reviews anything. The campaign goes live. You find out when finance asks why the ad account is drained.

```text
DIAGRAM: Ungated workflow failure at step 5
Caption: Without approval gates, the agent auto-advances through step 5 (set_ad_budget) with an incorrect $4,000 value, and all subsequent steps execute on that wrong number.
Nodes:
1. Steps 1 to 4: Execute correctly, no issues
2. Step 5 (set_ad_budget): Claude proposes $4,000/day (wrong), agent auto-advances
3. Steps 6 to 10: Execute using the wrong budget, campaign publishes
4. Human: Discovers the error 14 hours later from a billing alert
Flow:
- Steps 1 to 4 auto-advance without review
- Step 5 auto-advances with $4,000 daily budget (should be $40)
- Steps 6 to 10 proceed on the wrong number
- Human gets billing alert after money is spent
```

## The fix

The fix is already built into the implementation above. The `gated` flag on steps 4, 5, 7, and 9 is the control. But the critical piece is the approval check in `wait_for_approval`. That function refuses to return a result until the Cue API status flips to `approved`. No approval, no advancement. The state machine enforces this server-side, so even a buggy client cannot skip the gate. Here is the specific guard that makes the difference, extracted for clarity:

```python
# The core gate logic from agent.py, isolated
def wait_for_approval(workflow_id: str, step_name: str, proposed: str) -> str:
    """Block until a human approves or rejects. No timeout. The agent
    does not proceed without explicit human input at gated steps."""
    while True:
        resp = requests.get(
            f"{CUEAPI_BASE}/workflows/{workflow_id}/steps/{step_name}/status",
            headers=HEADERS
        )
        resp.raise_for_status()
        data = resp.json()

        if data["status"] == "approved":
            # Human may have corrected the value
            return data.get("corrected_action", proposed)

        if data["status"] == "rejected":
            raise SystemExit(f"Step '{step_name}' rejected. Workflow halted.")

        time.sleep(5)
```

The `corrected_action` field is what makes this more than a rubber stamp. The reviewer does not just say yes or no. They can rewrite the value. When Claude says $4,000, the reviewer types $40 and the workflow continues with the correct number.

## Fixed state

```text
DIAGRAM: Gated workflow catching the budget error at step 5
Caption: The state machine pauses at step 5, the human corrects $4,000 to $40, and all subsequent steps use the correct value.
Nodes:
1. Steps 1 to 3: Auto-advance, no review needed
2. Step 4 (select_image_keywords): GATED, human approves image search terms
3. Step 5 (set_ad_budget): GATED, Claude proposes $4,000, human corrects to $40
4. Step 6: Auto-advances with correct budget
5. Step 7 (configure_ad_targeting): GATED, human reviews targeting params
6. Step 8: Auto-advances
7. Step 9 (schedule_publication): GATED, human confirms publish date
8. Step 10: Auto-advances, workflow complete
Flow:
- Steps 1 to 3 run at full agent speed
- Step 4 pauses, human approves, agent continues
- Step 5 pauses, human sees $4,000, corrects to $40, agent continues with $40
- Step 6 uses corrected budget
- Step 7 pauses, human reviews targeting, approves
- Step 8 runs automatically
- Step 9 pauses, human confirms schedule
- Step 10 completes and outputs campaign summary
```

## After

You have a 10-step agent workflow. It researches, drafts, and configures at full speed through the low-risk steps. At step 5, it proposes a daily budget. The number looks wrong. You type the correct one and hit approve. The agent picks up the corrected value and finishes the remaining steps in under a minute. Total human time: about 90 seconds across four approval gates. Total risk: bounded to whatever you explicitly approved. The agent is still fast. It just cannot spend money or publish content without your sign-off. You sleep better.

## Takeaway

Not every step needs a gate. Gate the steps where a wrong decision costs real money, damages trust, or is hard to reverse. The pattern is: let the agent run free where mistakes are cheap and recoverable, force a human checkpoint where they are not. This applies to any multi-step agent workflow, not just publishing. Anywhere an LLM makes a decision that triggers an external side effect, put a gate.
