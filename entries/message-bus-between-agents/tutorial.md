## Opening thesis

Wire your agents together with a message bus instead of nested function calls. You will build a three-agent pipeline where each agent publishes structured output to the bus, and the next agent consumes it asynchronously. A message-bus-coordinated agent system handles stage failures, retries, and back-pressure correctly on every single run, where a human wiring the same coordination by hand with function calls would cut corners on the retry logic and regret it in production. The pipeline uses Inngest as the bus and the Anthropic API for each agent's reasoning.

## Before

You have three agents: a Research agent that fetches context, a Draft agent that writes copy, and a Review agent that scores quality. They call each other as nested functions. `review(draft(research(topic)))`. When the Draft agent hits a rate limit on the Anthropic API, the entire call stack throws. The Research agent's output is gone. You have no record of what succeeded. You add a try/catch. You forget to add a retry. You add a retry but forget exponential backoff. You add backoff but forget to cap it. You deploy on Friday. The pipeline fails at 2 AM, retries 47 times in a tight loop, burns through your API quota, and you wake up to a $200 invoice and zero completed drafts. The problem is not the agents. The problem is the wiring between them.

## Architecture

The replacement architecture puts Inngest between every agent. Each agent is an Inngest function triggered by an event. When the Research agent finishes, it sends an event called `research.completed`. The Draft agent listens for that event. When it finishes, it sends `draft.completed`. The Review agent listens for that. Each function has its own retry policy, its own timeout, and its own step-level state. A failure in the Draft agent does not touch the Research agent's output. Inngest stores every event payload, so you can replay any stage without re-running the ones before it.

```text
DIAGRAM: Message bus agent pipeline
Caption: Three Anthropic-powered agents coordinated through Inngest events
Nodes:
1. Trigger (HTTP request) - kicks off the pipeline with a topic
2. Inngest Event Bus - routes events between agents
3. Research Agent (Inngest fn) - calls Anthropic to gather context
4. Draft Agent (Inngest fn) - calls Anthropic to write copy from context
5. Review Agent (Inngest fn) - calls Anthropic to score and critique the draft
6. Result Store (step.run output) - persisted state per step
Flow:
- Trigger sends "pipeline.started" to Event Bus
- Event Bus triggers Research Agent on "pipeline.started"
- Research Agent sends "research.completed" with payload to Event Bus
- Event Bus triggers Draft Agent on "research.completed"
- Draft Agent sends "draft.completed" with payload to Event Bus
- Event Bus triggers Review Agent on "draft.completed"
- Review Agent sends "review.completed" with final output to Event Bus
```

## Step-by-step implementation

### 1. Set up the project and install dependencies

Create a new Node.js project. Install `inngest` for the event bus and the Anthropic SDK for agent reasoning.

```bash
mkdir agent-bus && cd agent-bus
npm init -y
npm install inngest@^3.0.0 @anthropic-ai/sdk@^0.30.0 express@^4.18.0
```

### 2. Set environment variables

You need two keys. Get your Anthropic API key from https://console.anthropic.com/settings/keys. Get your Inngest signing key and event key from https://app.inngest.com/env after creating an account. For local development, Inngest Dev Server does not require signing keys.

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
export INNGEST_EVENT_KEY="your-event-key"
export INNGEST_SIGNING_KEY="signkey-..."
```

### 3. Create the Anthropic client helper

Wrap the Anthropic call in a single function so every agent uses the same model and parameters. This file is `claude.js`.

```js
// claude.js
const Anthropic = require("@anthropic-ai/sdk");

const client = new Anthropic();

async function ask(systemPrompt, userMessage) {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: systemPrompt,
    messages: [{ role: "user", content: userMessage }],
  });
  return response.content[0].text;
}

module.exports = { ask };
```

### 4. Define the Research agent

The Research agent listens for `pipeline.started` events. It calls Claude to generate research notes on the provided topic. It then sends a `research.completed` event with those notes as the payload. The `step.run` wrapper means Inngest persists the output. If this function is retried, Inngest skips the completed step and does not call Claude again.

```js
// agents/research.js
const { inngest } = require("../inngestClient");
const { ask } = require("../claude");

const researchAgent = inngest.createFunction(
  { id: "research-agent", retries: 3 },
  { event: "pipeline.started" },
  async ({ event, step }) => {
    const notes = await step.run("gather-research", async () => {
      return ask(
        "You are a research assistant. Return 5 bullet points of key facts.",
        `Topic: ${event.data.topic}`
      );
    });

    await step.sendEvent("emit-research", {
      name: "research.completed",
      data: { topic: event.data.topic, notes },
    });

    return { notes };
  }
);

module.exports = { researchAgent };
```

### 5. Define the Draft agent

The Draft agent listens for `research.completed`. It receives the research notes in `event.data.notes` and asks Claude to write a short article. Same retry and step persistence pattern.

```js
// agents/draft.js
const { inngest } = require("../inngestClient");
const { ask } = require("../claude");

const draftAgent = inngest.createFunction(
  { id: "draft-agent", retries: 3 },
  { event: "research.completed" },
  async ({ event, step }) => {
    const draft = await step.run("write-draft", async () => {
      return ask(
        "You are a technical writer. Write a 200-word article using only the provided notes.",
        `Topic: ${event.data.topic}\n\nNotes:\n${event.data.notes}`
      );
    });

    await step.sendEvent("emit-draft", {
      name: "draft.completed",
      data: { topic: event.data.topic, draft },
    });

    return { draft };
  }
);

module.exports = { draftAgent };
```

### 6. Define the Review agent

The Review agent listens for `draft.completed`. It asks Claude to score the draft from 1 to 10 and list improvements. This is the final stage.

```js
// agents/review.js
const { inngest } = require("../inngestClient");
const { ask } = require("../claude");

const reviewAgent = inngest.createFunction(
  { id: "review-agent", retries: 3 },
  { event: "draft.completed" },
  async ({ event, step }) => {
    const review = await step.run("review-draft", async () => {
      return ask(
        "You are an editor. Score this draft 1-10. List three specific improvements.",
        `Draft:\n${event.data.draft}`
      );
    });

    await step.sendEvent("emit-review", {
      name: "review.completed",
      data: { topic: event.data.topic, review },
    });

    return { review };
  }
);

module.exports = { reviewAgent };
```

### 7. Create the Inngest client

This shared client is imported by every agent file. One client, one bus.

```js
// inngestClient.js
const { Inngest } = require("inngest");

const inngest = new Inngest({ id: "agent-bus" });

module.exports = { inngest };
```

### 8. Wire up the Express server and serve functions

Inngest functions are served over HTTP. This server registers all three agents and exposes the Inngest endpoint. It also exposes a POST route to trigger the pipeline.

```js
// server.js
const express = require("express");
const { serve } = require("inngest/express");
const { inngest } = require("./inngestClient");
const { researchAgent } = require("./agents/research");
const { draftAgent } = require("./agents/draft");
const { reviewAgent } = require("./agents/review");

const app = express();
app.use(express.json());

app.use("/api/inngest", serve({ client: inngest, functions: [researchAgent, draftAgent, reviewAgent] }));

app.post("/trigger", async (req, res) => {
  await inngest.send({ name: "pipeline.started", data: { topic: req.body.topic } });
  res.json({ status: "pipeline triggered" });
});

app.listen(3000, () => console.log("Server running on port 3000"));
```

### 9. Start the Inngest Dev Server and test locally

The Inngest Dev Server runs locally and provides the event bus, retry logic, and a dashboard. Start it in one terminal. Start your app in another. Then trigger the pipeline.

```bash
# Terminal 1: start Inngest Dev Server
npx inngest-cli@latest dev

# Terminal 2: start your app
node server.js

# Terminal 3: trigger the pipeline
curl -X POST http://localhost:3000/trigger \
  -H "Content-Type: application/json" \
  -d '{"topic": "how message buses improve agent reliability"}'
```

### 10. Observe the pipeline in the Inngest dashboard

Open http://localhost:8288 in your browser. You will see three functions listed. Click into any run to see step-level state: what succeeded, what was retried, what each step returned. Every event payload is visible. You can replay any function from the dashboard without touching your code.

```bash
# Open the local Inngest dashboard
open http://localhost:8288
```

## Breakage

Remove the `retries: 3` config from the Draft agent and kill the `step.run` wrapper so the Claude call happens outside Inngest's step tracking. Now when the Anthropic API returns a 529 (overloaded), the Draft agent fails permanently. Its output is lost. The Review agent never fires because no `draft.completed` event is emitted. The Research agent's work is preserved, but the pipeline is stuck. You check the Inngest dashboard and see the Draft function failed once with no retries. The payload from the Research agent sits in the event log, unclaimed. This is exactly the scenario that happens in production when you skip retry configuration: the first transient failure kills the pipeline.

```text
DIAGRAM: Pipeline breakage without retries
Caption: Draft agent fails on a transient API error and the pipeline halts
Nodes:
1. Research Agent - completes successfully
2. Inngest Event Bus - holds "research.completed" event
3. Draft Agent (no retries, no step tracking) - fails on 529 error
4. Review Agent - never triggered
Flow:
- Research Agent sends "research.completed" to Event Bus
- Event Bus triggers Draft Agent
- Draft Agent calls Anthropic API directly (no step.run)
- Anthropic returns 529 overloaded
- Draft Agent throws, no retry, no state saved
- No "draft.completed" event emitted
- Review Agent sits idle
```

## The fix

The fix is the code already written in Step 5. The `retries: 3` config tells Inngest to retry the function up to three times with exponential backoff. The `step.run` wrapper ensures that if the function is retried after a partial success, completed steps are not re-executed. Here is the critical section isolated, using the same variable names and file from Step 5:

```js
// In agents/draft.js, the function config and step wrapper are the fix.
// retries: 3 gives you automatic exponential backoff.
// step.run persists the output so retries skip completed work.

const draftAgent = inngest.createFunction(
  { id: "draft-agent", retries: 3 },
  { event: "research.completed" },
  async ({ event, step }) => {
    const draft = await step.run("write-draft", async () => {
      return ask(
        "You are a technical writer. Write a 200-word article using only the provided notes.",
        `Topic: ${event.data.topic}\n\nNotes:\n${event.data.notes}`
      );
    });

    await step.sendEvent("emit-draft", {
      name: "draft.completed",
      data: { topic: event.data.topic, draft },
    });

    return { draft };
  }
);
```

## Fixed state

```text
DIAGRAM: Pipeline with retries and step persistence
Caption: Draft agent retries on transient failure, resumes from last completed step
Nodes:
1. Research Agent - completes, output persisted
2. Inngest Event Bus - routes events, stores payloads
3. Draft Agent (retries: 3, step.run) - retries on 529, skips completed steps
4. Review Agent - triggered after draft.completed arrives
5. Step State Store - Inngest persists each step.run output
Flow:
- Research Agent sends "research.completed" to Event Bus
- Event Bus triggers Draft Agent
- Draft Agent calls Anthropic via step.run("write-draft")
- Anthropic returns 529 on first attempt
- Inngest retries Draft Agent with exponential backoff
- On retry, step.run("write-draft") re-executes (no prior success to skip)
- Anthropic returns 200 on second attempt
- step.run output persisted to Step State Store
- Draft Agent sends "draft.completed" to Event Bus
- Event Bus triggers Review Agent
- Review Agent completes, sends "review.completed"
```

## After

You have three agents: a Research agent, a Draft agent, and a Review agent. They communicate through events on a bus. When the Draft agent hits a rate limit, Inngest retries it with backoff. The Research agent's output is safe in the event log. The Review agent waits patiently for `draft.completed` and fires when it arrives. You check the dashboard and see the retry, the backoff delay, the successful completion. You deploy on Friday. The pipeline fails at 2 AM, retries three times over 90 seconds, succeeds on the third attempt, and finishes the run. You wake up to completed drafts and a normal invoice.

## Takeaway

The pattern is: put a durable event bus between every agent, make each agent a subscriber with its own retry policy, and wrap every side effect in a step that persists its output. This applies to any multi-agent system, not just content pipelines. When you move coordination out of your call stack and into infrastructure that was built for coordination, you stop writing retry logic by hand and start sleeping through the night.
