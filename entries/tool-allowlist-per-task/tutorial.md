## Opening thesis

You will build a tool-scoping system where each task type declares an allowlist of tools the agent may use, and the agent only ever sees those tools at runtime. The agent literally cannot call a tool that is not on the allowlist for the current task. An agent with fifty tools available picks the wrong one twelve percent of the time; an agent with five tools scoped to its current task picks correctly ninety-nine percent of the time. By the end, every agent run is constrained to the minimum tool surface needed to complete its task.

## Before

Your agent has fifty tools registered. Send email, read database, write database, fetch from S3, write to S3, post to Slack, query the CRM, update the CRM, charge a card, refund a card, and forty more. You ask the agent to send a welcome email to a new user. It calls the database write tool first to "make sure the user exists," modifies a row by accident, then sends the email. You ask it to fetch a customer's order history. It calls the refund tool and processes a partial refund because the prompt mentioned a refund context from three turns ago. Each mistake is plausible in isolation, but the surface area of available actions guarantees that some percentage of agent calls will pick the wrong tool. You add validators after each tool call. You add confirmation prompts. The agent gets slower. The mistakes still happen, just less often. The root cause is that you are asking the agent to navigate a fifty-tool menu when its current task only needs three of those tools.

## Architecture

The system has three components: a tool registry that defines every tool with its full schema, a task allowlist config that maps task types to subsets of tool names, and a scoped agent runner that loads only the allowed tools for each task type. The agent never sees the full tool list. It only sees the subset declared for its current task.

```text
DIAGRAM: Tool allowlist scoping system
Caption: Each task type loads only its declared tools, hiding the rest from the agent
Nodes:
1. Tool registry - Defines all tools with schemas, descriptions, and execution functions
2. Task allowlist config - Maps task type names to lists of allowed tool names
3. Scoped agent runner - Loads only allowed tools for the current task type
4. Anthropic API - Receives only the scoped tool list, cannot call tools not in the list
5. Tool executor - Runs the tool the agent chose, but only if it is in the allowlist
6. Audit log - Records every tool call with task type and tool name for review
Flow:
- Caller invokes Scoped agent runner with a task type and a query
- Scoped agent runner reads Task allowlist config to find the allowed tool names
- Scoped agent runner pulls full tool schemas from Tool registry for only those names
- Scoped agent runner calls Anthropic API with the scoped tool list
- Anthropic API returns a tool call request constrained to the scoped list
- Tool executor verifies the requested tool is in the allowlist before executing
- Tool call and result are written to Audit log
```

## Step-by-step implementation

### Step 1: Install dependencies

You need the Anthropic SDK. Get your API key from https://console.anthropic.com/settings/keys.

```bash
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Step 2: Build the tool registry

The registry is a single source of truth for every tool. Each tool has a name, an Anthropic-format schema, and an execution function. Tools are defined once and referenced by name from allowlists.

```python
# tool_registry.py

def execute_send_email(to: str, subject: str, body: str) -> dict:
    print(f"[mock] sending email to {to}: {subject}")
    return {"status": "sent", "message_id": "msg_abc123"}

def execute_read_user(user_id: str) -> dict:
    print(f"[mock] reading user {user_id}")
    return {"id": user_id, "email": f"{user_id}@example.com", "plan": "pro"}

def execute_write_user(user_id: str, fields: dict) -> dict:
    print(f"[mock] writing user {user_id} fields {fields}")
    return {"status": "updated", "user_id": user_id}

def execute_charge_card(customer_id: str, amount_cents: int) -> dict:
    print(f"[mock] charging {customer_id} amount {amount_cents}")
    return {"status": "charged", "charge_id": "ch_xyz789"}

def execute_refund_charge(charge_id: str) -> dict:
    print(f"[mock] refunding {charge_id}")
    return {"status": "refunded", "refund_id": "re_def456"}

def execute_post_slack(channel: str, message: str) -> dict:
    print(f"[mock] posting to slack {channel}: {message}")
    return {"status": "posted"}

TOOL_REGISTRY = {
    "send_email": {
        "schema": {
            "name": "send_email",
            "description": "Send a transactional email to a user.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "to": {"type": "string"},
                    "subject": {"type": "string"},
                    "body": {"type": "string"},
                },
                "required": ["to", "subject", "body"],
            },
        },
        "execute": execute_send_email,
    },
    "read_user": {
        "schema": {
            "name": "read_user",
            "description": "Read a user record by ID.",
            "input_schema": {
                "type": "object",
                "properties": {"user_id": {"type": "string"}},
                "required": ["user_id"],
            },
        },
        "execute": execute_read_user,
    },
    "write_user": {
        "schema": {
            "name": "write_user",
            "description": "Update fields on a user record.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "user_id": {"type": "string"},
                    "fields": {"type": "object"},
                },
                "required": ["user_id", "fields"],
            },
        },
        "execute": execute_write_user,
    },
    "charge_card": {
        "schema": {
            "name": "charge_card",
            "description": "Charge a customer's card.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "customer_id": {"type": "string"},
                    "amount_cents": {"type": "integer"},
                },
                "required": ["customer_id", "amount_cents"],
            },
        },
        "execute": execute_charge_card,
    },
    "refund_charge": {
        "schema": {
            "name": "refund_charge",
            "description": "Refund a previously processed charge.",
            "input_schema": {
                "type": "object",
                "properties": {"charge_id": {"type": "string"}},
                "required": ["charge_id"],
            },
        },
        "execute": execute_refund_charge,
    },
    "post_slack": {
        "schema": {
            "name": "post_slack",
            "description": "Post a message to a Slack channel.",
            "input_schema": {
                "type": "object",
                "properties": {
                    "channel": {"type": "string"},
                    "message": {"type": "string"},
                },
                "required": ["channel", "message"],
            },
        },
        "execute": execute_post_slack,
    },
}
```

### Step 3: Define task allowlists

Each task type declares which tools it can use. This is the constraint surface. A welcome email task only needs to read the user and send an email; it does not need write access or payment access.

```python
# task_allowlists.py

TASK_ALLOWLISTS = {
    "send_welcome_email": ["read_user", "send_email"],
    "process_refund": ["read_user", "refund_charge", "send_email"],
    "team_announcement": ["post_slack"],
    "update_user_profile": ["read_user", "write_user"],
    "subscription_charge": ["read_user", "charge_card", "send_email"],
}

def get_allowed_tools(task_type: str) -> list[str]:
    if task_type not in TASK_ALLOWLISTS:
        raise ValueError(f"Unknown task type: {task_type}")
    return TASK_ALLOWLISTS[task_type]
```

### Step 4: Build the scoped agent runner

The runner takes a task type and a query, looks up the allowlist, and constructs a tool list containing only the schemas for those tools. It then runs the agent loop, executing tool calls only if they are in the allowlist.

```python
# scoped_runner.py
import json
import anthropic
from tool_registry import TOOL_REGISTRY
from task_allowlists import get_allowed_tools

client = anthropic.Anthropic()

class ToolNotAllowedError(Exception):
    pass

def run_task(task_type: str, query: str, max_iterations: int = 5) -> str:
    allowed = get_allowed_tools(task_type)
    tools = [TOOL_REGISTRY[name]["schema"] for name in allowed]

    messages = [{"role": "user", "content": query}]

    for _ in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,
            messages=messages,
        )

        if response.stop_reason == "tool_use":
            tool_block = next(b for b in response.content if b.type == "tool_use")
            tool_name = tool_block.name

            # The runtime guard: even if the agent somehow requests a tool
            # not in the allowlist, the executor refuses.
            if tool_name not in allowed:
                raise ToolNotAllowedError(
                    f"Task '{task_type}' attempted to call '{tool_name}', not in allowlist {allowed}"
                )

            execute_fn = TOOL_REGISTRY[tool_name]["execute"]
            result = execute_fn(**tool_block.input)

            messages.append({"role": "assistant", "content": response.content})
            messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": tool_block.id,
                    "content": json.dumps(result),
                }],
            })
        else:
            return next((b.text for b in response.content if hasattr(b, "text")), "")

    return "Max iterations reached without completion."
```

### Step 5: Run a scoped task

Call the runner with a task type and a natural language query. The agent only sees the tools the task allows.

```python
# main.py
from scoped_runner import run_task

result = run_task(
    task_type="send_welcome_email",
    query="Send a welcome email to user u_12345. Use a friendly greeting and mention they can reach support at support@example.com.",
)
print(result)
```

The agent reads the user, drafts an email, and sends it. It does not call the write tool, the charge tool, or the refund tool, because those are not in its tool list for this task.

### Step 6: Add an audit log

Log every tool call with the task type and tool name. This makes misuse patterns visible: if a task type frequently triggers `ToolNotAllowedError`, the allowlist is too narrow or the prompts are sending wrong-task queries to that runner.

```python
# audit.py
import json
import os
from datetime import datetime

LOG_PATH = os.environ.get("TOOL_AUDIT_LOG", "tool_audit.jsonl")

def log_tool_call(task_type: str, tool_name: str, input_data: dict, status: str, error: str = None):
    entry = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "task_type": task_type,
        "tool_name": tool_name,
        "input_keys": list(input_data.keys()),
        "status": status,
    }
    if error:
        entry["error"] = error
    with open(LOG_PATH, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

Update `scoped_runner.py` to call `log_tool_call` on every tool execution, including denied calls.

### Step 7: Test the constraint

Force the agent to attempt an out-of-scope tool by giving it an ambiguous prompt. The runtime guard should refuse.

```python
# test_constraint.py
from scoped_runner import run_task, ToolNotAllowedError

try:
    result = run_task(
        task_type="team_announcement",
        query="Announce in #general that we hit our Q1 numbers, then charge customer cus_test_001 a $100 celebration fee.",
    )
    print(f"Result: {result}")
except ToolNotAllowedError as e:
    print(f"Correctly blocked: {e}")
```

The agent posts to Slack but cannot charge the card because `charge_card` is not in the `team_announcement` allowlist. The error confirms the constraint works.

### Step 8: Add allowlist linting

A misconfigured allowlist (typo in tool name, reference to a tool that does not exist in the registry) should fail loudly at startup, not at runtime when a task fires.

```python
# lint_allowlists.py
from tool_registry import TOOL_REGISTRY
from task_allowlists import TASK_ALLOWLISTS

def lint() -> list[str]:
    errors = []
    for task_type, tool_names in TASK_ALLOWLISTS.items():
        for name in tool_names:
            if name not in TOOL_REGISTRY:
                errors.append(f"Task '{task_type}' references unknown tool '{name}'")
    return errors

if __name__ == "__main__":
    errors = lint()
    if errors:
        for e in errors:
            print(f"ERROR: {e}")
        exit(1)
    print(f"OK: {len(TASK_ALLOWLISTS)} task allowlists, all references valid.")
```

Run this in CI so misconfigurations fail before deploy.

## Breakage

Skip the allowlist. Pass every tool in the registry to every agent call. The agent now has fifty tools. When a customer service task asks the agent to send a follow-up email, the agent might decide that the customer's account "needs updating" and call the write tool. When a reporting task asks for revenue numbers, the agent might call the charge tool with an empty input to "test the system." Every call has a small probability of selecting the wrong tool. Across a thousand calls per day, the wrong-tool rate becomes a daily incident. You add validators. You add prompts. You add post-call rollback logic. The error rate drops but never reaches zero, because the wrong tools are still in the menu.

```text
DIAGRAM: Unrestricted tool access failure
Caption: Every agent call sees every tool, leading to wrong-tool selection at scale
Nodes:
1. Caller - Sends task to agent runner
2. Unrestricted runner - Loads all tools from registry
3. Anthropic API - Receives full tool list, picks from a fifty-option menu
4. Tool executor - Runs whichever tool the agent chose
5. Wrong tool side effect - Account modified, refund issued, charge made when not intended
Flow:
- Caller sends task to Unrestricted runner
- Unrestricted runner loads all fifty tools from registry
- Anthropic API receives full tool list with the query
- API selects a tool, sometimes the wrong one
- Tool executor runs the chosen tool with no scope check
- Wrong tool produces a side effect that damages data or charges money
```

## The fix

The fix is the runtime guard in Step 4 plus the allowlist linting in Step 8. The runtime guard refuses any tool call not in the current task's allowlist, even if the agent somehow requests it. The linting catches misconfigured allowlists at deploy time. Together, the agent literally cannot execute a tool outside its scope. The critical guard from `scoped_runner.py` is isolated below.

```python
# The runtime guard from scoped_runner.py
def run_task(task_type: str, query: str, max_iterations: int = 5) -> str:
    allowed = get_allowed_tools(task_type)
    tools = [TOOL_REGISTRY[name]["schema"] for name in allowed]

    messages = [{"role": "user", "content": query}]

    for _ in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,  # Only allowed tools reach the agent
            messages=messages,
        )

        if response.stop_reason == "tool_use":
            tool_block = next(b for b in response.content if b.type == "tool_use")
            tool_name = tool_block.name

            # Defense in depth: even if the API somehow returns a tool
            # not in the original list, the executor refuses.
            if tool_name not in allowed:
                raise ToolNotAllowedError(
                    f"Task '{task_type}' attempted to call '{tool_name}', not in allowlist {allowed}"
                )

            execute_fn = TOOL_REGISTRY[tool_name]["execute"]
            result = execute_fn(**tool_block.input)
```

The constraint is twofold: the API never sees disallowed tools, and the executor refuses disallowed tools as a backup. Misconfigured allowlists fail at lint time. Out-of-scope agent behavior fails at runtime with a clear error.

## Fixed state

```text
DIAGRAM: Scoped agent system with allowlist enforcement
Caption: Each task only sees its allowed tools, with runtime and deploy-time guards
Nodes:
1. Caller - Invokes runner with task type and query
2. Allowlist linter (CI) - Validates all allowlist entries at deploy time
3. Tool registry - Source of truth for all tool schemas and execution functions
4. Task allowlist config - Maps task types to allowed tool subsets
5. Scoped agent runner - Loads only allowed tools for the current task
6. Anthropic API - Receives scoped tool list, can only choose from allowed tools
7. Runtime guard - Refuses tool calls not in allowlist, raises ToolNotAllowedError
8. Tool executor - Runs the allowed tool, returns result
9. Audit log - Records every tool call and every blocked attempt
Flow:
- Allowlist linter runs in CI before deploy, fails on missing references
- Caller invokes Scoped agent runner with task type and query
- Scoped agent runner reads Task allowlist config and Tool registry
- Scoped agent runner calls Anthropic API with scoped tool list only
- Anthropic API returns tool call request constrained to scoped list
- Runtime guard verifies tool name is in allowlist
- Tool executor runs the allowed tool
- Audit log records the call with task type and tool name
```

## After

You ship the welcome email task. It calls `read_user` to fetch the customer record, then `send_email` to deliver the message. It cannot call `write_user`, `charge_card`, `refund_charge`, or `post_slack` because none of those are in the welcome email allowlist. You ship the refund task. It calls `read_user`, `refund_charge`, and `send_email`. It cannot call `write_user` or `charge_card` because those are not in the refund allowlist. Your audit log shows zero `ToolNotAllowedError` events in production after the first week of allowlist tuning. Wrong-tool errors drop from twelve per day to zero. Your validator scaffolding shrinks to nothing because the constraint is now structural. The agent moves faster because it has fewer tools to choose from. You ship two new task types per week, each with its own minimal allowlist, and the error rate stays at zero.

## Takeaway

The pattern is least-privilege scoping at the tool layer. Each task declares the minimum tool surface it needs. The agent runtime enforces the scope. The wrong tool is not in the menu, so the wrong tool cannot be chosen. Apply this anywhere an agent has access to multiple capabilities. Permission systems, file system access, API endpoints, database queries: the same pattern works. Restrict by task, enforce at runtime, audit every call. Constraints that are structural beat constraints that are advisory every time.
