## Opening thesis

You will build a wrapper around the Anthropic API that validates every response against a JSON schema, automatically retries with the validation error appended to the prompt when validation fails, and surfaces a hard error after a bounded number of failed attempts. An agent with strict schema validation retries until it produces valid output, which a human reviewing free-form output would catch by reading every response and re-prompting manually. By the end, your downstream code can rely on the agent producing structured output that matches the contract every time.

## Before

You ask the agent to extract structured data from a customer email: name, order number, issue category, urgency. The agent returns prose. You add a system prompt that says "return JSON." The agent returns JSON wrapped in markdown fences. You strip the fences. The agent returns JSON with a hallucinated extra field called "notes" that your downstream code does not expect. You add stricter prompts. The agent returns JSON with `urgency: "high"` instead of `urgency: 5` because the prompt was unclear. You add examples. The agent occasionally returns valid JSON, occasionally returns malformed JSON, occasionally returns JSON with the right shape but wrong types. Your downstream code crashes one in five times. You add try-except blocks and fallback values. The fallbacks corrupt your data pipeline silently. The root problem: you are asking the agent for structure but accepting whatever it produces.

## Architecture

The system has three components: a JSON schema definition for the expected output, a validator that checks responses against the schema, and a retry loop that re-prompts the agent with the schema and the validation error if the response is invalid. Validation happens before the response leaves the wrapper. Downstream code only ever sees output that has passed validation.

```text
DIAGRAM: Schema-validated agent wrapper
Caption: Validation gate enforces structured output with bounded retry on failure
Nodes:
1. Caller - Provides query and JSON schema
2. Validated agent wrapper - Constructs prompt with schema, calls API, validates response
3. Anthropic API - Returns response that may or may not match schema
4. JSON parser - Attempts to parse response as JSON, may fail
5. Schema validator - Checks parsed JSON against the provided schema
6. Retry composer - On validation failure, builds a new prompt with the error and retries
7. Hard failure - After N retries, raises an error and surfaces the last response
8. Validated output - Returned to Caller only after schema validation passes
Flow:
- Caller invokes Validated agent wrapper with query and JSON schema
- Validated agent wrapper builds a prompt that includes the schema and instructs JSON-only output
- Anthropic API returns a response
- JSON parser attempts to parse the response
- If parse fails: Retry composer builds a new prompt with the parse error
- If parse succeeds: Schema validator checks the structure
- If validation fails: Retry composer builds a new prompt with the validation error
- If validation succeeds: Validated output returns to Caller
- After N failed retries: Hard failure raises an error
```

## Step-by-step implementation

### Step 1: Install dependencies

You need the Anthropic SDK and the `jsonschema` library for validation.

```bash
pip install anthropic jsonschema
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Step 2: Define an example schema

Schemas use the standard JSON Schema format. The schema below describes a customer issue extraction task with four required fields and explicit types and enums.

```python
# schemas.py

CUSTOMER_ISSUE_SCHEMA = {
    "type": "object",
    "properties": {
        "customer_name": {"type": "string"},
        "order_number": {"type": "string", "pattern": "^ORD-[0-9]{6}$"},
        "category": {
            "type": "string",
            "enum": ["billing", "shipping", "product", "account", "other"],
        },
        "urgency": {"type": "integer", "minimum": 1, "maximum": 5},
    },
    "required": ["customer_name", "order_number", "category", "urgency"],
    "additionalProperties": False,
}
```

The `additionalProperties: false` clause is the constraint that catches hallucinated extra fields.

### Step 3: Build the prompt composer

The composer takes a query and a schema and constructs a system prompt that includes the schema and explicit formatting rules. The schema embedded in the prompt acts as both contract and example.

```python
# prompt_composer.py
import json

def build_system_prompt(schema: dict) -> str:
    return f"""You produce structured JSON output matching the schema below. No prose, no markdown fences, no commentary.

Schema:
{json.dumps(schema, indent=2)}

Rules:
- Return ONLY a single JSON object that validates against the schema.
- Do not add fields that are not in the schema.
- Do not omit required fields.
- Match the types exactly. If the schema says "integer", return a number, not a string.
- For enum fields, use only values from the enum list."""
```

### Step 4: Build the JSON parser with cleanup

LLMs sometimes wrap JSON in markdown fences despite instructions. The parser strips fences before parsing, then attempts to extract a JSON object even if there is leading or trailing whitespace.

```python
# json_parser.py
import json
import re

def parse_response(text: str):
    cleaned = text.strip()

    # Strip markdown fences if present
    if cleaned.startswith("```"):
        cleaned = re.sub(r"^```(?:json)?\s*", "", cleaned)
        cleaned = re.sub(r"\s*```$", "", cleaned)

    # If there is leading/trailing prose, try to extract the JSON object
    match = re.search(r"\{.*\}", cleaned, re.DOTALL)
    if match:
        cleaned = match.group(0)

    return json.loads(cleaned)
```

### Step 5: Build the schema validator

The validator uses the `jsonschema` library to check parsed JSON against the schema. If validation fails, it returns the error message in a form the retry loop can use as feedback.

```python
# schema_validator.py
from jsonschema import validate, ValidationError

class ValidationFailed(Exception):
    def __init__(self, message: str, path: str):
        super().__init__(message)
        self.message = message
        self.path = path

def validate_against_schema(data, schema: dict) -> None:
    try:
        validate(instance=data, schema=schema)
    except ValidationError as e:
        path = ".".join(str(p) for p in e.path) if e.path else "root"
        raise ValidationFailed(message=e.message, path=path)
```

### Step 6: Build the validated agent wrapper

The wrapper combines the prompt composer, the JSON parser, the schema validator, and a retry loop. On parse or validation failure, it composes a new prompt that includes the original query, the previous response, and the specific error, then retries up to a maximum number of attempts.

```python
# validated_agent.py
import json
import anthropic
from prompt_composer import build_system_prompt
from json_parser import parse_response
from schema_validator import validate_against_schema, ValidationFailed

client = anthropic.Anthropic()

class AgentValidationError(Exception):
    pass

def query_with_schema(query: str, schema: dict, max_attempts: int = 3) -> dict:
    system = build_system_prompt(schema)
    user_message = query
    last_response_text = None
    last_error = None

    for attempt in range(1, max_attempts + 1):
        if attempt > 1 and last_response_text is not None:
            user_message = (
                f"{query}\n\n"
                f"Your previous attempt:\n{last_response_text}\n\n"
                f"Validation error: {last_error}\n\n"
                f"Return a corrected JSON object that matches the schema."
            )

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=system,
            messages=[{"role": "user", "content": user_message}],
        )
        text = response.content[0].text
        last_response_text = text

        try:
            parsed = parse_response(text)
        except json.JSONDecodeError as e:
            last_error = f"JSON parse failure: {e}"
            continue

        try:
            validate_against_schema(parsed, schema)
            return parsed
        except ValidationFailed as e:
            last_error = f"Schema validation failed at '{e.path}': {e.message}"
            continue

    raise AgentValidationError(
        f"Agent failed to produce valid output after {max_attempts} attempts. "
        f"Last error: {last_error}. Last response: {last_response_text}"
    )
```

### Step 7: Use the validated wrapper in your application

Call `query_with_schema` with a query and a schema. The function returns a parsed and validated dict, ready for downstream code.

```python
# main.py
from validated_agent import query_with_schema, AgentValidationError
from schemas import CUSTOMER_ISSUE_SCHEMA

email_text = """
Hi there, my name is Sarah Chen and I placed order ORD-348201 last week.
The package arrived today but the wrong product was inside. I need this
resolved by Friday because it's a gift. Please help.
"""

try:
    result = query_with_schema(
        query=f"Extract the structured customer issue from this email:\n\n{email_text}",
        schema=CUSTOMER_ISSUE_SCHEMA,
    )
    print(f"Customer: {result['customer_name']}")
    print(f"Order: {result['order_number']}")
    print(f"Category: {result['category']}")
    print(f"Urgency: {result['urgency']}")
except AgentValidationError as e:
    print(f"Failed to extract: {e}")
```

### Step 8: Add retry telemetry

Track how often each schema requires retries. Schemas that need frequent retries indicate either a too-strict schema or a poorly-worded prompt. Either way, the data tells you where to improve.

```python
# telemetry.py
import json
import os
from datetime import datetime

LOG_PATH = os.environ.get("VALIDATION_LOG", "validation_telemetry.jsonl")

def log_attempt(schema_name: str, attempt: int, success: bool, error: str = None):
    entry = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "schema_name": schema_name,
        "attempt": attempt,
        "success": success,
    }
    if error:
        entry["error"] = error
    with open(LOG_PATH, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

Update `validated_agent.py` to call `log_attempt` after each attempt, passing a `schema_name` parameter through `query_with_schema`.

## Breakage

Skip the schema validator. Trust the agent to return JSON because the prompt says to return JSON. Most of the time, it works. Then a customer email arrives with formatting that confuses the agent. The agent returns JSON with `urgency: "very high"` instead of an integer. Your downstream code expects an integer and crashes. You add a try-except. The except block logs the failure and substitutes a default urgency of 3. Your support team gets a flood of urgency-3 tickets that should have been urgency-5. The data is wrong. The pipeline keeps running. You only notice when the customer who was about to churn gets de-prioritized for a week.

```text
DIAGRAM: Unvalidated output failure mode
Caption: Free-form agent output causes silent downstream corruption
Nodes:
1. Caller - Sends query to agent
2. Unvalidated wrapper - Calls API, returns raw response
3. Anthropic API - Returns text that may or may not be valid JSON
4. Downstream parser - Attempts to parse, sometimes succeeds, sometimes fails
5. Try-except fallback - Substitutes default values when parsing fails
6. Corrupted pipeline - Receives mixed valid and default data, processes it as real
Flow:
- Caller sends query to Unvalidated wrapper
- Unvalidated wrapper calls Anthropic API
- Anthropic API returns text response of varying quality
- Downstream parser parses successfully sometimes, fails sometimes
- Try-except fallback substitutes default values on failure
- Corrupted pipeline cannot distinguish real data from fallbacks
- Bad decisions made on corrupted data
```

## The fix

The fix is the validated wrapper from Step 6. The retry loop is the mechanism: when the agent produces output that fails validation, the wrapper does not return the bad output and does not silently substitute defaults. It re-prompts the agent with the specific validation error and the previous response, and the agent corrects itself. The critical part of the loop is isolated below.

```python
# The retry mechanism from validated_agent.py
def query_with_schema(query: str, schema: dict, max_attempts: int = 3) -> dict:
    system = build_system_prompt(schema)
    user_message = query
    last_response_text = None
    last_error = None

    for attempt in range(1, max_attempts + 1):
        # On retry: include the previous response and the validation error.
        # The agent learns from its own mistake within the same call.
        if attempt > 1 and last_response_text is not None:
            user_message = (
                f"{query}\n\n"
                f"Your previous attempt:\n{last_response_text}\n\n"
                f"Validation error: {last_error}\n\n"
                f"Return a corrected JSON object that matches the schema."
            )

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=system,
            messages=[{"role": "user", "content": user_message}],
        )
        text = response.content[0].text
        last_response_text = text

        try:
            parsed = parse_response(text)
            validate_against_schema(parsed, schema)
            return parsed  # Only return validated output
        except (json.JSONDecodeError, ValidationFailed) as e:
            last_error = str(e)

    # Hard failure after max attempts: never return invalid output.
    raise AgentValidationError(
        f"Failed after {max_attempts} attempts. Last error: {last_error}."
    )
```

The loop never returns invalid output. Either the agent converges to valid output within `max_attempts`, or the wrapper raises an error. There is no silent fallback. Downstream code only ever receives data that matches the schema.

## Fixed state

```text
DIAGRAM: Validated agent wrapper with retry loop
Caption: Every output is parsed, validated, and corrected before reaching downstream code
Nodes:
1. Caller - Provides query and schema
2. Prompt composer - Builds system prompt embedding the schema
3. Anthropic API - Returns response
4. JSON parser - Attempts to parse, captures error if fails
5. Schema validator - Checks parsed object against schema, captures error if fails
6. Retry composer - On failure, builds new prompt with previous response and error
7. Validated output - Returned only after passing both parse and schema checks
8. Hard failure - Raised after N attempts, never silent
9. Telemetry log - Records every attempt for tuning
Flow:
- Caller invokes wrapper with query and schema
- Prompt composer builds system prompt with schema embedded
- Anthropic API returns response
- JSON parser attempts to parse the response
- If parse fails: Retry composer builds new prompt and loop continues
- If parse succeeds: Schema validator runs
- If validation fails: Retry composer builds new prompt with the specific error
- If validation succeeds: Validated output returns to Caller
- After N failed attempts: Hard failure raised
- Every attempt logged to Telemetry log
```

## After

You call `query_with_schema` with a customer email and the issue extraction schema. The agent returns JSON. The validator checks it. Valid. Your downstream code receives a dict with the right shape, the right types, and no extra fields. You call it a thousand times across a day. The retry counter shows 940 first-attempt successes, 55 second-attempt successes, 5 third-attempt successes, and zero hard failures. Your downstream pipeline never crashes. Your data quality is consistent because every record passes the same structural check. You add a new schema for ticket prioritization. You add another for refund eligibility scoring. Each one is reliable from day one because the wrapper enforces the contract. The agent is fast and structured. Your code stops needing defensive parsing.

## Takeaway

The pattern is contract enforcement at the boundary. Define the contract as a schema. Validate every output against the schema. Retry with the error as feedback when validation fails. Never silently substitute. Apply this to any LLM output that downstream code depends on for structure: extraction, classification, decision-making, tool selection. The retry loop is what turns a probabilistic system into a reliable one. The schema is what makes the contract explicit. Together, they let you treat agent output as if it were the output of a well-typed function.
