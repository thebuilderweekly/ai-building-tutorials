## Opening thesis

You will build a Python wrapper around the Anthropic API that tracks cumulative token spend per session and per day, throttles when approaching limits, downgrades to a cheaper model when over a soft threshold, and hard-stops before real damage. An agent looping on a malformed input can burn through a month's API budget in 20 minutes; a budget-enforced agent stops itself before the bill arrives. The agent beats a human at this task because it checks the meter on every single call, something no person staring at a dashboard at 3 a.m. will do reliably.

## Before

You ship an agent on Friday afternoon. It processes incoming support tickets, calls Claude to draft responses, and queues them for review. At 2:14 a.m., a webhook delivers a ticket with a malformed Unicode body. Your retry logic catches the parse error, re-prompts Claude, gets the same error, and retries. By 6 a.m. the loop has fired 4,200 requests against `claude-sonnet-4-20250514`. Your Anthropic dashboard shows $1,137.42 in usage. Your CFO sends a one-line email: "Explain this." You have no explanation, because nothing in your code was watching the bill.

## Architecture

The system wraps every Anthropic API call in a budget tracker. The tracker reads from a local JSON ledger that records cumulative cost per session and per calendar day. Before each call, the tracker checks the ledger against three thresholds: a soft limit (switch to a cheaper model), a throttle limit (add a delay between calls), and a hard limit (refuse the call entirely). After each call, the tracker writes the actual token cost back to the ledger.

```text
DIAGRAM: Budget-enforced agent architecture
Caption: Every API call passes through a budget gate before reaching Anthropic.
Nodes:
1. Agent loop - generates prompts from incoming tasks
2. BudgetGate - reads ledger, enforces thresholds, selects model
3. Ledger (JSON file) - stores cumulative cost per session and per day
4. Anthropic API - processes the prompt if the gate allows it
5. Response handler - parses response, logs token usage back to ledger
Flow:
- Agent loop sends prompt to BudgetGate
- BudgetGate reads Ledger to check cumulative spend
- If under soft limit: BudgetGate forwards prompt to Anthropic API with requested model
- If over soft limit but under hard limit: BudgetGate downgrades model and/or adds delay
- If over hard limit: BudgetGate raises BudgetExhaustedError, call never reaches Anthropic
- Anthropic API returns response to Response handler
- Response handler computes cost from usage tokens, writes to Ledger
```

## Step-by-step implementation

### 1. Install the Anthropic SDK

You need the official Python SDK. Nothing else.

```bash
pip install anthropic
```

### 2. Set your API key

Get your key from [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys). Export it so every script in this tutorial can read it.

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

### 3. Define the pricing table and budget config

Pricing changes. Hard-code current prices and update them when Anthropic publishes new rates. These numbers reflect the published per-token prices as of April 2026. The config sets three thresholds in USD: soft (switch model), throttle (add delay), and hard (stop entirely).

```python
# budget_config.py

# Prices per token in USD
MODEL_PRICES = {
    "claude-sonnet-4-20250514": {"input": 3.00 / 1_000_000, "output": 15.00 / 1_000_000},
    "claude-haiku-4-20250514": {"input": 0.80 / 1_000_000, "output": 4.00 / 1_000_000},
}

DEFAULT_MODEL = "claude-sonnet-4-20250514"
DOWNGRADE_MODEL = "claude-haiku-4-20250514"

# Budget thresholds in USD
SESSION_SOFT_LIMIT = 5.00    # switch to cheaper model
SESSION_THROTTLE_LIMIT = 8.00  # add 2-second delay between calls
SESSION_HARD_LIMIT = 10.00   # stop entirely

DAILY_SOFT_LIMIT = 25.00
DAILY_THROTTLE_LIMIT = 40.00
DAILY_HARD_LIMIT = 50.00
```

### 4. Build the ledger

The ledger is a JSON file on disk. It stores cost per session ID and per calendar date. Reading and writing use file locks to handle concurrent processes. This is the source of truth for all budget decisions.

```python
# ledger.py
import json
import os
import fcntl
from datetime import date

LEDGER_PATH = os.environ.get("BUDGET_LEDGER_PATH", "/tmp/agent_budget_ledger.json")

def _read_ledger():
    if not os.path.exists(LEDGER_PATH):
        return {"sessions": {}, "daily": {}}
    with open(LEDGER_PATH, "r") as f:
        fcntl.flock(f, fcntl.LOCK_SH)
        data = json.load(f)
        fcntl.flock(f, fcntl.LOCK_UN)
    return data

def _write_ledger(data):
    with open(LEDGER_PATH, "w") as f:
        fcntl.flock(f, fcntl.LOCK_EX)
        json.dump(data, f, indent=2)
        fcntl.flock(f, fcntl.LOCK_UN)

def get_session_spend(session_id: str) -> float:
    ledger = _read_ledger()
    return ledger["sessions"].get(session_id, 0.0)

def get_daily_spend() -> float:
    ledger = _read_ledger()
    today = date.today().isoformat()
    return ledger["daily"].get(today, 0.0)

def record_spend(session_id: str, cost: float):
    ledger = _read_ledger()
    today = date.today().isoformat()
    ledger["sessions"][session_id] = ledger["sessions"].get(session_id, 0.0) + cost
    ledger["daily"][today] = ledger["daily"].get(today, 0.0) + cost
    _write_ledger(ledger)
```

### 5. Build the budget gate

The gate sits between your agent and the Anthropic client. It checks the ledger before every call. It returns a decision: proceed (with which model), throttle (with a delay), or halt. This is the core of the system.

```python
# budget_gate.py
import time
from budget_config import (
    DEFAULT_MODEL, DOWNGRADE_MODEL,
    SESSION_SOFT_LIMIT, SESSION_THROTTLE_LIMIT, SESSION_HARD_LIMIT,
    DAILY_SOFT_LIMIT, DAILY_THROTTLE_LIMIT, DAILY_HARD_LIMIT,
)
from ledger import get_session_spend, get_daily_spend


class BudgetExhaustedError(Exception):
    pass


def check_budget(session_id: str, requested_model: str = DEFAULT_MODEL) -> str:
    """Returns the model to use. Raises BudgetExhaustedError if hard limit hit."""
    session_spend = get_session_spend(session_id)
    daily_spend = get_daily_spend()

    # Hard limits: stop entirely
    if session_spend >= SESSION_HARD_LIMIT:
        raise BudgetExhaustedError(
            f"Session {session_id} hit hard limit: ${session_spend:.2f} / ${SESSION_HARD_LIMIT:.2f}"
        )
    if daily_spend >= DAILY_HARD_LIMIT:
        raise BudgetExhaustedError(
            f"Daily hard limit reached: ${daily_spend:.2f} / ${DAILY_HARD_LIMIT:.2f}"
        )

    model = requested_model

    # Soft limits: downgrade model
    if session_spend >= SESSION_SOFT_LIMIT or daily_spend >= DAILY_SOFT_LIMIT:
        model = DOWNGRADE_MODEL

    # Throttle limits: add delay
    if session_spend >= SESSION_THROTTLE_LIMIT or daily_spend >= DAILY_THROTTLE_LIMIT:
        time.sleep(2.0)

    return model
```

### 6. Build the wrapped client

This function replaces your raw `client.messages.create` calls. It calls the budget gate, makes the API request, computes the cost from the response's usage field, records it to the ledger, and returns the response. Every call is metered.

```python
# guarded_client.py
import anthropic
from budget_config import MODEL_PRICES
from budget_gate import check_budget
from ledger import record_spend

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env


def guarded_create(session_id: str, messages: list, model: str = None, max_tokens: int = 1024):
    from budget_config import DEFAULT_MODEL
    requested = model or DEFAULT_MODEL

    # Gate decides model or raises
    actual_model = check_budget(session_id, requested)

    response = client.messages.create(
        model=actual_model,
        max_tokens=max_tokens,
        messages=messages,
    )

    # Compute cost
    prices = MODEL_PRICES.get(actual_model, MODEL_PRICES[DEFAULT_MODEL])
    input_cost = response.usage.input_tokens * prices["input"]
    output_cost = response.usage.output_tokens * prices["output"]
    total_cost = input_cost + output_cost

    record_spend(session_id, total_cost)

    return response, {"model_used": actual_model, "call_cost": total_cost}
```

### 7. Write the agent loop

This simulates a real agent processing tasks. It calls `guarded_create` in a loop. When the budget gate raises `BudgetExhaustedError`, the loop exits cleanly. No infinite retries. No surprise bills.

```python
# agent.py
import sys
from guarded_client import guarded_create
from budget_gate import BudgetExhaustedError

SESSION_ID = "ticket-responder-001"

tasks = [
    "Summarize this support ticket: 'My login is broken since Tuesday.'",
    "Summarize this support ticket: 'Billing page shows wrong currency.'",
    "Summarize this support ticket: 'Cannot export CSV from dashboard.'",
]

for i, task in enumerate(tasks):
    print(f"Processing task {i + 1}/{len(tasks)}...")
    try:
        response, meta = guarded_create(
            session_id=SESSION_ID,
            messages=[{"role": "user", "content": task}],
        )
        print(f"  Model: {meta['model_used']}, Cost: ${meta['call_cost']:.6f}")
        print(f"  Response: {response.content[0].text[:120]}...")
    except BudgetExhaustedError as e:
        print(f"  BUDGET STOP: {e}")
        sys.exit(0)

print("All tasks complete.")
```

### 8. Test the hard limit

Set the session hard limit to $0.01 temporarily, then run the agent. You should see it stop after one or two calls. This proves the gate works before you deploy to production.

```python
# test_hard_limit.py
import os
os.environ["BUDGET_LEDGER_PATH"] = "/tmp/test_budget_ledger.json"

# Remove stale test ledger
import pathlib
pathlib.Path("/tmp/test_budget_ledger.json").unlink(missing_ok=True)

# Patch limits for testing
import budget_config
budget_config.SESSION_HARD_LIMIT = 0.01
budget_config.SESSION_SOFT_LIMIT = 0.005
budget_config.SESSION_THROTTLE_LIMIT = 0.008

from guarded_client import guarded_create
from budget_gate import BudgetExhaustedError

for i in range(10):
    try:
        resp, meta = guarded_create(
            session_id="test-session",
            messages=[{"role": "user", "content": "Say hello in one word."}],
            max_tokens=16,
        )
        print(f"Call {i+1}: model={meta['model_used']}, cost=${meta['call_cost']:.6f}")
    except BudgetExhaustedError as e:
        print(f"Call {i+1}: STOPPED. {e}")
        break
```

### 9. Add a daily spend report

A budget system you never look at is a budget system you forget to update. This script reads the ledger and prints a summary. Run it from cron or a CI job every morning.

```python
# report.py
import json
from ledger import _read_ledger
from budget_config import DAILY_HARD_LIMIT

ledger = _read_ledger()
print("=== Daily Spend Report ===")
for day, amount in sorted(ledger.get("daily", {}).items()):
    pct = (amount / DAILY_HARD_LIMIT) * 100
    print(f"  {day}: ${amount:.4f} ({pct:.1f}% of ${DAILY_HARD_LIMIT:.2f} daily limit)")

print("\n=== Session Spend ===")
for sid, amount in sorted(ledger.get("sessions", {}).items()):
    print(f"  {sid}: ${amount:.4f}")
```

## Breakage

Skip the budget gate and call `client.messages.create` directly. Your agent hits a malformed input at 2 a.m. The retry logic fires the same failing prompt over and over. Each call costs between $0.02 and $0.30 depending on context length. After 4,000 retries you have spent over a thousand dollars. Nothing in the code noticed because nothing was counting.

```text
DIAGRAM: Failure without budget enforcement
Caption: The retry loop calls the API directly with no cost check.
Nodes:
1. Agent loop - retries on error
2. Anthropic API - charges per token on every call, successful or not
3. Invoice - grows unchecked
Flow:
- Agent loop sends malformed prompt to Anthropic API
- Anthropic API returns error or partial response
- Agent loop catches exception, retries immediately
- Anthropic API charges for tokens consumed
- Loop repeats thousands of times
- Invoice reaches $1,000+ before anyone wakes up
```

## The fix

The fix is already in the code above: `guarded_create` replaces every direct `client.messages.create` call. But the specific piece that stops the overnight disaster is the hard limit check in `check_budget`. Here is that function isolated, with an added logging line so you can see every block event in your logs.

```python
# budget_gate.py (updated check_budget with logging)
import time
import logging
from budget_config import (
    DEFAULT_MODEL, DOWNGRADE_MODEL,
    SESSION_SOFT_LIMIT, SESSION_THROTTLE_LIMIT, SESSION_HARD_LIMIT,
    DAILY_SOFT_LIMIT, DAILY_THROTTLE_LIMIT, DAILY_HARD_LIMIT,
)
from ledger import get_session_spend, get_daily_spend

logger = logging.getLogger("budget_gate")
logging.basicConfig(level=logging.INFO)


class BudgetExhaustedError(Exception):
    pass


def check_budget(session_id: str, requested_model: str = DEFAULT_MODEL) -> str:
    session_spend = get_session_spend(session_id)
    daily_spend = get_daily_spend()

    if session_spend >= SESSION_HARD_LIMIT:
        logger.warning("HARD LIMIT HIT session=%s spend=%.4f limit=%.2f", session_id, session_spend, SESSION_HARD_LIMIT)
        raise BudgetExhaustedError(
            f"Session {session_id} hit hard limit: ${session_spend:.2f} / ${SESSION_HARD_LIMIT:.2f}"
        )
    if daily_spend >= DAILY_HARD_LIMIT:
        logger.warning("HARD LIMIT HIT daily spend=%.4f limit=%.2f", daily_spend, DAILY_HARD_LIMIT)
        raise BudgetExhaustedError(
            f"Daily hard limit reached: ${daily_spend:.2f} / ${DAILY_HARD_LIMIT:.2f}"
        )

    model = requested_model

    if session_spend >= SESSION_SOFT_LIMIT or daily_spend >= DAILY_SOFT_LIMIT:
        logger.info("Soft limit reached, downgrading to %s", DOWNGRADE_MODEL)
        model = DOWNGRADE_MODEL

    if session_spend >= SESSION_THROTTLE_LIMIT or daily_spend >= DAILY_THROTTLE_LIMIT:
        logger.info("Throttle limit reached, adding 2s delay")
        time.sleep(2.0)

    return model
```

## Fixed state

```text
DIAGRAM: Budget-enforced agent under retry storm
Caption: The budget gate blocks calls once the hard limit is reached, even during a retry loop.
Nodes:
1. Agent loop - retries on error
2. BudgetGate - checks ledger before every call
3. Ledger (JSON file) - cumulative cost record
4. Anthropic API - only receives calls that pass the gate
5. Logger - records every downgrade, throttle, and block event
Flow:
- Agent loop sends prompt to BudgetGate
- BudgetGate reads Ledger
- First N calls: cost is under soft limit, calls proceed to Anthropic API with default model
- After soft limit: BudgetGate switches to cheaper model, calls still proceed
- After throttle limit: BudgetGate adds 2-second delay, calls still proceed slowly
- After hard limit: BudgetGate raises BudgetExhaustedError, agent loop exits
- No further calls reach Anthropic API
- Total spend stays under the hard limit
```

## After

You ship the agent on Friday afternoon. It processes incoming support tickets, calls Claude to draft responses, and queues them for review. At 2:14 a.m., the same malformed Unicode ticket arrives. Your retry logic fires. After three retries, the session spend crosses the $5 soft limit and the gate downgrades to Haiku. After five more retries it crosses the $8 throttle limit and each retry now takes two extra seconds. After two more retries it crosses the $10 hard limit. The gate raises `BudgetExhaustedError`. The agent logs the event and exits. Your Monday morning spend report shows $10.03 for that session. Your CFO never writes the email.

## Takeaway

The pattern is a pre-call gate with a post-call meter. Every external API call passes through a function that can deny it, downgrade it, or slow it down based on cumulative recorded spend. Apply this pattern to any API with per-call pricing: LLMs, search APIs, image generators, speech-to-text services. The gate does not need to be complex. It needs to exist.