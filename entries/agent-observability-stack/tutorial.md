## Opening thesis

You will build an observability layer around a multi-step AI agent powered by the Anthropic API. The agent will emit structured logs, distributed traces, and per-step latency metrics for every workflow run. An agent without observability is a black box; an agent with structured logs, traces, and per-step latency metrics tells you exactly where it failed and why, before the user noticed. A human reviewing print-statement output across twelve steps takes hours to find a root cause. The instrumented agent surfaces the broken step in seconds.

## Before

Your agent failed at 3 a.m. You woke up to a Slack message from the on-call engineer: "customer export job produced empty output." You open the code. It is a twelve-step pipeline: fetch user data, summarize it with Claude, validate the summary, enrich with metadata, format as PDF, upload to S3, notify the customer, and five more steps you barely remember writing. You add `print("got here 1")` to the top of each function. You rerun the job. It passes this time. You add more prints. You rerun. It fails again, somewhere different. Two hours later you find the bug: a malformed API response in step seven that your code swallowed silently. You fix it, remove the print statements, push, and hope it holds. You have no idea if it will.

## Architecture

The system wraps every step of your agent pipeline in a tracing context. Each step emits a structured JSON log entry with a trace ID, step name, input hash, output hash, latency, and status. A lightweight collector writes these entries to a local JSONL file (swap this for any log sink in production). A query script reads the file and reconstructs the full trace for any run.

```text
DIAGRAM: Observability-Instrumented Agent Pipeline
Caption: Shows how each agent step emits structured events into a shared trace collected by a local JSONL sink.
Nodes:
1. Agent Runner - Orchestrates the multi-step workflow and assigns a trace ID per run.
2. Step Wrapper - Decorates each step function, captures timing, input/output hashes, errors.
3. Anthropic API - External LLM call made during summarization and validation steps.
4. Structured Logger - Writes JSON event per step to the JSONL sink.
5. JSONL Sink - Local file (or remote log store) holding all trace events.
6. Trace Query Script - Reads the sink, filters by trace ID, prints the full timeline.
Flow:
- Agent Runner starts a run, generates a trace ID, calls each step through the Step Wrapper.
- Step Wrapper records start time, calls the actual step function, records end time and status.
- Step Wrapper sends the structured event to the Structured Logger.
- Structured Logger appends a JSON line to the JSONL Sink.
- Trace Query Script reads the JSONL Sink and reconstructs any run on demand.
```

## Step-by-step implementation

### 1. Set up the project and dependencies

Create a directory and install the two packages you need: the Anthropic Python SDK and a hashing library (hashlib is in the standard library). Get your API key from https://console.anthropic.com/settings/keys and export it.

```bash
mkdir agent-observability && cd agent-observability
python3 -m venv venv && source venv/bin/activate
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-..."  # paste your real key here
```

### 2. Define the structured event schema

Create a file called `events.py`. This module defines the shape of every event your agent will emit. Every field is typed. No freeform strings for status: only "ok" or "error".

```python
# events.py
import json
import time
import hashlib
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class StepEvent:
    trace_id: str
    step_index: int
    step_name: str
    status: str  # "ok" or "error"
    start_epoch: float
    end_epoch: float
    latency_ms: float
    input_hash: str
    output_hash: Optional[str]
    error_message: Optional[str]

    def to_json(self) -> str:
        return json.dumps(asdict(self))

def compute_hash(data: str) -> str:
    return hashlib.sha256(data.encode()).hexdigest()[:12]
```

### 3. Build the structured logger

Create `logger.py`. It appends one JSON line per event to a local file. In production, swap `open()` for a call to your log aggregator's HTTP endpoint. The file path is configurable via an environment variable.

```python
# logger.py
import os
from events import StepEvent

LOG_PATH = os.environ.get("AGENT_LOG_PATH", "traces.jsonl")

def emit(event: StepEvent) -> None:
    with open(LOG_PATH, "a") as f:
        f.write(event.to_json() + "\n")
```

### 4. Create the step wrapper decorator

Create `wrapper.py`. This decorator captures timing, hashes inputs and outputs, catches exceptions, and emits a `StepEvent` through the logger. Every step function you decorate gets observability for free.

```python
# wrapper.py
import time
import traceback
from events import StepEvent, compute_hash
import logger

def traced_step(step_index: int, step_name: str, trace_id_holder: dict):
    def decorator(fn):
        def wrapper(input_text: str) -> str:
            start = time.time()
            input_h = compute_hash(input_text)
            try:
                result = fn(input_text)
                end = time.time()
                event = StepEvent(
                    trace_id=trace_id_holder["id"],
                    step_index=step_index,
                    step_name=step_name,
                    status="ok",
                    start_epoch=start,
                    end_epoch=end,
                    latency_ms=round((end - start) * 1000, 2),
                    input_hash=input_h,
                    output_hash=compute_hash(result),
                    error_message=None,
                )
                logger.emit(event)
                return result
            except Exception as e:
                end = time.time()
                event = StepEvent(
                    trace_id=trace_id_holder["id"],
                    step_index=step_index,
                    step_name=step_name,
                    status="error",
                    start_epoch=start,
                    end_epoch=end,
                    latency_ms=round((end - start) * 1000, 2),
                    input_hash=input_h,
                    output_hash=None,
                    error_message=str(e),
                )
                logger.emit(event)
                raise
        return wrapper
    return decorator
```

### 5. Write the agent steps

Create `agent.py`. This file defines a six-step agent. Steps 2 and 4 call the Anthropic API via the [SDK](https://docs.anthropic.com/en/api). The other steps simulate data processing. Each step is a plain function decorated with `traced_step`.

```python
# agent.py
import uuid
import anthropic
from wrapper import traced_step

trace_id_holder = {"id": ""}
client = anthropic.Anthropic()

@traced_step(1, "fetch_data", trace_id_holder)
def fetch_data(input_text: str) -> str:
    return f"Raw data for: {input_text}"

@traced_step(2, "summarize", trace_id_holder)
def summarize(input_text: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=256,
        messages=[{"role": "user", "content": f"Summarize this in one sentence: {input_text}"}],
    )
    return response.content[0].text

@traced_step(3, "validate_length", trace_id_holder)
def validate_length(input_text: str) -> str:
    if len(input_text) > 2000:
        raise ValueError("Summary too long")
    return input_text

@traced_step(4, "enrich_metadata", trace_id_holder)
def enrich_metadata(input_text: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=128,
        messages=[{"role": "user", "content": f"Add category tags to this text as a comma-separated prefix: {input_text}"}],
    )
    return response.content[0].text

@traced_step(5, "format_output", trace_id_holder)
def format_output(input_text: str) -> str:
    return f"=== REPORT ===\n{input_text}\n=== END ==="

@traced_step(6, "deliver", trace_id_holder)
def deliver(input_text: str) -> str:
    return f"Delivered: {input_text[:50]}..."

def run(user_input: str) -> str:
    trace_id_holder["id"] = str(uuid.uuid4())
    print(f"Trace ID: {trace_id_holder['id']}")
    data = fetch_data(user_input)
    summary = summarize(data)
    validated = validate_length(summary)
    enriched = enrich_metadata(validated)
    formatted = format_output(enriched)
    result = deliver(formatted)
    return result

if __name__ == "__main__":
    output = run("Q1 2026 revenue numbers for ACME Corp")
    print(output)
```

### 6. Run the agent and inspect the raw trace

Execute the agent. Then look at the JSONL file it produced. Each line is one step event.

```bash
python agent.py
cat traces.jsonl | python -m json.tool --json-lines
```

### 7. Build the trace query script

Create `query_trace.py`. It reads the JSONL file, filters by trace ID, and prints a timeline with latency and status for each step. This is your debugging entrypoint.

```python
# query_trace.py
import sys
import json

def query(trace_id: str, log_path: str = "traces.jsonl") -> None:
    events = []
    with open(log_path) as f:
        for line in f:
            event = json.loads(line)
            if event["trace_id"] == trace_id:
                events.append(event)
    events.sort(key=lambda e: e["step_index"])
    print(f"Trace: {trace_id}")
    print(f"Steps: {len(events)}")
    print("-" * 72)
    total_ms = 0.0
    for e in events:
        total_ms += e["latency_ms"]
        err = f"  ERROR: {e['error_message']}" if e["error_message"] else ""
        print(
            f"  [{e['step_index']}] {e['step_name']:<20} "
            f"{e['latency_ms']:>8.1f}ms  {e['status']}{err}"
        )
    print("-" * 72)
    print(f"  Total: {total_ms:.1f}ms")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python query_trace.py <trace_id>")
        sys.exit(1)
    query(sys.argv[1])
```

### 8. Query a specific trace

Grab the trace ID from the agent's stdout and pass it to the query script.

```bash
python query_trace.py "paste-your-trace-id-here"
```

You will see output like this:

```text
Trace: a1b2c3d4-...
Steps: 6
------------------------------------------------------------------------
  [1] fetch_data                0.1ms  ok
  [2] summarize              1203.4ms  ok
  [3] validate_length           0.0ms  ok
  [4] enrich_metadata         987.2ms  ok
  [5] format_output             0.0ms  ok
  [6] deliver                   0.0ms  ok
------------------------------------------------------------------------
  Total: 2190.7ms
```

### 9. Add a metrics summary script

Create `metrics.py`. It reads all traces and computes p50, p95, and p99 latency per step name. This tells you which steps are slow across many runs, not just one.

```python
# metrics.py
import json
import statistics

def report(log_path: str = "traces.jsonl") -> None:
    step_latencies: dict[str, list[float]] = {}
    with open(log_path) as f:
        for line in f:
            e = json.loads(line)
            step_latencies.setdefault(e["step_name"], []).append(e["latency_ms"])
    print(f"{'Step':<20} {'Count':>6} {'p50':>10} {'p95':>10} {'p99':>10}")
    print("-" * 60)
    for name, lats in sorted(step_latencies.items()):
        lats_sorted = sorted(lats)
        n = len(lats_sorted)
        p50 = statistics.median(lats_sorted)
        p95 = lats_sorted[int(n * 0.95)] if n >= 20 else lats_sorted[-1]
        p99 = lats_sorted[int(n * 0.99)] if n >= 100 else lats_sorted[-1]
        print(f"{name:<20} {n:>6} {p50:>9.1f}ms {p95:>9.1f}ms {p99:>9.1f}ms")

if __name__ == "__main__":
    report()
```

```bash
python metrics.py
```

## Breakage

Remove the `traced_step` decorators. Run the agent. It still works, until it doesn't. When step 4 (`enrich_metadata`) starts returning malformed JSON from the Anthropic API, `format_output` receives garbage, and `deliver` silently ships a broken report. You have no record of what `enrich_metadata` returned. You have no latency data showing that step 4 took 14 seconds instead of its usual 1 second (a clear sign of a retry storm). You have no trace ID to correlate this run with the customer complaint. You are back to print statements.

```text
DIAGRAM: Unobserved Failure Path
Caption: Shows how a silent failure in step 4 propagates undetected to delivery.
Nodes:
1. Agent Runner - Runs all steps sequentially.
2. fetch_data - Step 1, succeeds.
3. summarize - Step 2, succeeds.
4. validate_length - Step 3, succeeds.
5. enrich_metadata - Step 4, returns malformed output silently.
6. format_output - Step 5, wraps garbage without checking.
7. deliver - Step 6, ships broken report.
8. No logs - Nothing recorded. No trace. No latency.
Flow:
- Agent Runner calls steps 1 through 6 in order.
- enrich_metadata returns bad data but does not raise an exception.
- format_output and deliver proceed normally with corrupted input.
- No event is emitted anywhere. The failure is invisible.
```

## The fix

The traced_step decorator already captures errors that raise exceptions. But the real danger is the silent corruption: a step that returns bad data without raising. Add an output validator callback to the wrapper. The validator checks the output after each step, and if it fails, the wrapper records the event as an error and raises. This closes the gap between "the step returned" and "the step returned something useful."

```python
# Add this to wrapper.py, replacing the existing traced_step function.

def traced_step(step_index: int, step_name: str, trace_id_holder: dict, validator=None):
    def decorator(fn):
        def wrapper(input_text: str) -> str:
            start = time.time()
            input_h = compute_hash(input_text)
            try:
                result = fn(input_text)
                if validator and not validator(result):
                    raise ValueError(f"Output validation failed for {step_name}")
                end = time.time()
                event = StepEvent(
                    trace_id=trace_id_holder["id"],
                    step_index=step_index,
                    step_name=step_name,
                    status="ok",
                    start_epoch=start,
                    end_epoch=end,
                    latency_ms=round((end - start) * 1000, 2),
                    input_hash=input_h,
                    output_hash=compute_hash(result),
                    error_message=None,
                )
                logger.emit(event)
                return result
            except Exception as e:
                end = time.time()
                event = StepEvent(
                    trace_id=trace_id_holder["id"],
                    step_index=step_index,
                    step_name=step_name,
                    status="error",
                    start_epoch=start,
                    end_epoch=end,
                    latency_ms=round((end - start) * 1000, 2),
                    input_hash=input_h,
                    output_hash=None,
                    error_message=str(e),
                )
                logger.emit(event)
                raise
        return wrapper
    return decorator
```

Now update the `enrich_metadata` step in `agent.py` to use a validator:

```python
def has_tags(text: str) -> bool:
    return "," in text and len(text) < 1000

@traced_step(4, "enrich_metadata", trace_id_holder, validator=has_tags)
def enrich_metadata(input_text: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=128,
        messages=[{"role": "user", "content": f"Add category tags to this text as a comma-separated prefix: {input_text}"}],
    )
    return response.content[0].text
```

## Fixed state

```text
DIAGRAM: Observed and Validated Pipeline
Caption: Shows the full pipeline with tracing, latency capture, and output validation at every step.
Nodes:
1. Agent Runner - Orchestrates steps, assigns trace ID.
2. Step Wrapper with Validator - Captures timing, hashes, runs output validation, emits events.
3. Anthropic API - External LLM calls in steps 2 and 4.
4. Structured Logger - Writes JSON events to the JSONL sink.
5. JSONL Sink - Stores all events for querying.
6. Trace Query Script - Reconstructs any run by trace ID.
7. Metrics Script - Computes p50/p95/p99 latency per step across runs.
Flow:
- Agent Runner calls each step through the Step Wrapper with Validator.
- If the validator rejects the output, the wrapper records an error event and halts the run.
- All events (ok and error) are written to the JSONL Sink via the Structured Logger.
- Trace Query Script filters events by trace ID for single-run debugging.
- Metrics Script aggregates latency across all runs for performance monitoring.
```

## After

Your agent failed at 3 a.m. again. You woke up, opened the trace query script, pasted the trace ID from the alert, and saw it immediately: step 4, `enrich_metadata`, status "error", latency 14,230ms, error message "Output validation failed for enrich_metadata." The input hash matched a known edge case you had been meaning to handle. You wrote a guard clause, pushed the fix, and went back to sleep. Total time from alert to fix: nine minutes. The metrics script later confirmed the p95 latency for `enrich_metadata` had been climbing for three days, a drift you would have caught earlier if you had been checking. Now you check every morning. It takes thirty seconds.

## Takeaway

The pattern is: wrap every step in a decorator that emits a structured event with a trace ID, timing, and content hash before returning. Validate outputs inside the wrapper, not after. When you treat observability as infrastructure that ships with the agent (not as something you add during an incident), debugging becomes a query instead of a guessing game.