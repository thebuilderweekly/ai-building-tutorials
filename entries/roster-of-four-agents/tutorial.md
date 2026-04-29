## Opening thesis

You will run a roster of four agents in a single workflow without the wheels coming off, taking a raw product brief and producing a finished technical spec with architecture decisions, an API contract, and a test plan. An agent team with well-defined handoffs produces the same multi-step result end-to-end in one minute that a four-person human team takes a full meeting to coordinate, and the agent team doesn't need a meeting. Each agent receives typed input, produces typed output, and passes a clean baton to the next without the chain coming apart.

## Before

You chained four Claude calls together in a script. The first agent wrote a decent requirements summary. The second agent received the full chat history plus the summary plus its own system prompt, and it started hallucinating requirements that were never in the brief. By agent three the context window was half-consumed by repeated instructions. Agent four produced an API contract that contradicted the architecture from agent two, because agent four never saw agent two's output cleanly. It saw a blob. You tried adding "IGNORE PREVIOUS INSTRUCTIONS" markers. That made it worse. You tried trimming context manually. That broke the chain. You gave up and scheduled a meeting with three other humans. The meeting took fifty-five minutes and you still left with action items nobody tracked.

## Architecture

The system uses [Inngest](https://www.inngest.com/docs) as a step-based orchestrator for a four-step workflow. Each step calls the [Anthropic Messages API](https://docs.anthropic.com/en/api/messages) with a dedicated system prompt and a typed payload. The orchestrator handles retries, step isolation, and observability. No agent sees another agent's system prompt. Each agent sees only the structured output of the previous agent.

```text
DIAGRAM: Four-agent spec-writing pipeline
Caption: Linear workflow where each agent receives only the typed output of the previous step
Nodes:
1. Orchestrator Function - runs the four steps in sequence and stores intermediate results
2. Agent 1 (Requirements) - extracts structured requirements from a raw brief
3. Agent 2 (Architecture) - produces architecture decisions from requirements
4. Agent 3 (API Contract) - generates OpenAPI paths from architecture decisions
5. Agent 4 (Test Plan) - writes test cases from the API contract
Flow:
- Raw brief enters the orchestrator via event trigger
- Orchestrator calls Agent 1, receives RequirementsOutput
- Orchestrator calls Agent 2 with RequirementsOutput, receives ArchitectureOutput
- Orchestrator calls Agent 3 with ArchitectureOutput, receives APIContractOutput
- Orchestrator calls Agent 4 with APIContractOutput, receives TestPlanOutput
- Orchestrator returns the combined typed result
```

## Step-by-step implementation

### 1. Set up the project and install dependencies

Create a new Node.js project. Install the Inngest SDK and the Anthropic SDK. Both are on npm.

```bash
mkdir spec-pipeline && cd spec-pipeline
npm init -y
npm install inngest @anthropic-ai/sdk typescript tsx @types/node
npx tsc --init --target es2022 --module nodenext --moduleResolution nodenext --outDir dist --strict true
```

### 2. Define the typed interfaces for each handoff

Every handoff between agents is a TypeScript interface. This is the load-bearing wall. If the types are sloppy, the agents drift. Put these in a shared file so the orchestrator and every agent caller reference the same shapes.

```ts
// src/types.ts
export interface RequirementsOutput {
  functional: string[];
  nonFunctional: string[];
  constraints: string[];
  actors: string[];
}

export interface ArchitectureOutput {
  components: { name: string; responsibility: string }[];
  dataFlow: string[];
  techChoices: { category: string; choice: string; rationale: string }[];
}

export interface APIContractOutput {
  endpoints: {
    method: string;
    path: string;
    summary: string;
    requestBody: string;
    responseShape: string;
  }[];
}

export interface TestPlanOutput {
  cases: {
    endpoint: string;
    scenario: string;
    expectedStatus: number;
    assertion: string;
  }[];
}
```

### 3. Build the agent caller with strict JSON output

Wrap the Anthropic call in a function that enforces JSON parsing. If the model returns text that is not valid JSON, throw immediately. Do not pass garbage downstream. Set the ANTHROPIC_API_KEY environment variable using a key from https://console.anthropic.com/settings/keys.

```ts
// src/callAgent.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function callAgent<T>(
  systemPrompt: string,
  userPayload: string
): Promise<T> {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 4096,
    system: systemPrompt,
    messages: [{ role: "user", content: userPayload }],
  });

  const text =
    response.content[0].type === "text" ? response.content[0].text : "";

  // Strip markdown fences if the model wraps JSON in them
  const cleaned = text.replace(/^```json\n?/, "").replace(/\n?```$/, "");

  const parsed: T = JSON.parse(cleaned);
  return parsed;
}
```

### 4. Write the four system prompts

Each prompt tells the agent exactly what JSON shape to produce. The prompt never references another agent. It only describes input and output. Store them in one file for easy auditing.

```ts
// src/prompts.ts
export const REQUIREMENTS_PROMPT = `You are a requirements analyst.
You receive a raw product brief as plain text.
Extract structured requirements.
Return ONLY a JSON object with these keys:
- functional: string array of functional requirements
- nonFunctional: string array of non-functional requirements
- constraints: string array of business or technical constraints
- actors: string array of user types or system actors
No commentary. No markdown. Only valid JSON.`;

export const ARCHITECTURE_PROMPT = `You are a software architect.
You receive a JSON object containing requirements (functional, nonFunctional, constraints, actors).
Produce architecture decisions.
Return ONLY a JSON object with these keys:
- components: array of {name, responsibility}
- dataFlow: string array describing data movement between components
- techChoices: array of {category, choice, rationale}
No commentary. No markdown. Only valid JSON.`;

export const API_CONTRACT_PROMPT = `You are an API designer.
You receive a JSON object containing architecture decisions (components, dataFlow, techChoices).
Design the REST API contract.
Return ONLY a JSON object with this key:
- endpoints: array of {method, path, summary, requestBody, responseShape}
No commentary. No markdown. Only valid JSON.`;

export const TEST_PLAN_PROMPT = `You are a QA engineer.
You receive a JSON object containing API endpoints.
Write test cases for each endpoint.
Return ONLY a JSON object with this key:
- cases: array of {endpoint, scenario, expectedStatus, assertion}
No commentary. No markdown. Only valid JSON.`;
```

### 5. Create the orchestrator client and function

Each step is isolated. If Agent 3 fails, only Agent 3 retries. The outputs of Agents 1 and 2 are already stored. This matters because Anthropic calls cost money and time. You do not want to re-run the whole chain for a transient 529.

```ts
// src/inngest.ts
import { Inngest } from "inngest";

export const inngest = new Inngest({ id: "spec-pipeline" });
```

### 6. Define the workflow function with four steps

Each `step.run` call is isolated. The orchestrator serializes the return value. The next step receives only what you pass it. This is the typed handoff that keeps agents from contaminating each other.

```ts
// src/workflow.ts
import { inngest } from "./inngest";
import { callAgent } from "./callAgent";
import {
  REQUIREMENTS_PROMPT,
  ARCHITECTURE_PROMPT,
  API_CONTRACT_PROMPT,
  TEST_PLAN_PROMPT,
} from "./prompts";
import type {
  RequirementsOutput,
  ArchitectureOutput,
  APIContractOutput,
  TestPlanOutput,
} from "./types";

export const specPipeline = inngest.createFunction(
  { id: "spec-pipeline", retries: 2 },
  { event: "spec/brief.submitted" },
  async ({ event, step }) => {
    const brief: string = event.data.brief;

    const requirements = await step.run("extract-requirements", () =>
      callAgent<RequirementsOutput>(REQUIREMENTS_PROMPT, brief)
    );

    const architecture = await step.run("design-architecture", () =>
      callAgent<ArchitectureOutput>(
        ARCHITECTURE_PROMPT,
        JSON.stringify(requirements)
      )
    );

    const apiContract = await step.run("define-api-contract", () =>
      callAgent<APIContractOutput>(
        API_CONTRACT_PROMPT,
        JSON.stringify(architecture)
      )
    );

    const testPlan = await step.run("write-test-plan", () =>
      callAgent<TestPlanOutput>(
        TEST_PLAN_PROMPT,
        JSON.stringify(apiContract)
      )
    );

    return { requirements, architecture, apiContract, testPlan };
  }
);
```

### 7. Serve the function locally

The orchestrator provides a dev server that runs functions locally. Start it alongside your serve endpoint.

```ts
// src/serve.ts
import { serve } from "inngest/node";
import { inngest } from "./inngest";
import { specPipeline } from "./workflow";
import { createServer } from "node:http";

const handler = serve({ client: inngest, functions: [specPipeline] });

const server = createServer(handler);
server.listen(3000, () => {
  console.log("Inngest serve endpoint running on port 3000");
});
```

```bash
# Terminal 1: start the serve endpoint
ANTHROPIC_API_KEY=your_key_here npx tsx src/serve.ts

# Terminal 2: start the Inngest dev server (install once: npx inngest-cli@latest)
npx inngest-cli@latest dev -u http://localhost:3000/api/inngest
```

### 8. Trigger the workflow with a test brief

Send an event to the dev server. The brief is plain text. Agent 1 handles the messy input. Every subsequent agent only sees structured JSON.

```bash
curl -X POST http://localhost:8288/e/spec-pipeline \
  -H "Content-Type: application/json" \
  -d '{
    "name": "spec/brief.submitted",
    "data": {
      "brief": "Build a booking system for a coworking space. Members reserve desks by the hour. Admins manage floor plans. Payments through Stripe. Must handle 500 concurrent users. GDPR compliant."
    }
  }'
```

### 9. Inspect intermediate results in the orchestrator dashboard

Open http://localhost:8288 in your browser. Click into the function run. Each step shows its return value as JSON. You can verify that Agent 1 produced clean requirements before Agent 2 ever touched them. If Agent 3 returned a malformed contract, you see exactly where the chain broke without re-running anything.

```bash
# You can also fetch a run's step output via the orchestrator REST API
curl http://localhost:8288/v1/events?name=spec/brief.submitted | jq '.data[0]'
```

## Breakage

Remove the `JSON.parse` validation from `callAgent`. Let each agent receive the raw text response of the previous agent, system prompt fragments included. Agent 2 gets a string that starts with "Here are the requirements:" followed by JSON. Agent 2 then echoes that preamble in its own output. Agent 3 receives a response that contains two layers of conversational wrapping around the actual architecture JSON. By Agent 4, the payload is mostly prose about JSON rather than JSON itself. The test plan is either a hallucination or a parse error. You cannot tell which step introduced the corruption without manually reading all four outputs, because nothing validated the intermediate shapes.

```text
DIAGRAM: Failure mode without typed validation
Caption: Raw text passes between agents, accumulating preamble and malformed wrapping
Nodes:
1. Agent 1 - returns "Here are the requirements: {json}"
2. Agent 2 - receives prose + json, echoes prose, wraps its own output in prose
3. Agent 3 - receives double-wrapped prose, produces unreliable output
4. Agent 4 - receives triple-wrapped prose, throws parse error or hallucinates
Flow:
- Agent 1 output has conversational preamble
- Agent 2 input is contaminated, output adds more preamble
- Agent 3 input is doubly contaminated
- Agent 4 fails or produces garbage
```

## The fix

The `callAgent` function already strips markdown fences and calls `JSON.parse`. But a stricter fix validates the parsed object against the expected shape. Add a validation layer that checks for required keys before returning. This catches the case where the model returns valid JSON that does not match the expected interface.

```ts
// Add to src/callAgent.ts, below the existing callAgent function

export function validateKeys<T extends Record<string, unknown>>(
  parsed: T,
  requiredKeys: string[]
): T {
  const missing = requiredKeys.filter((k) => !(k in parsed));
  if (missing.length > 0) {
    throw new Error(
      `Agent output missing required keys: ${missing.join(", ")}. ` +
      `Received keys: ${Object.keys(parsed).join(", ")}`
    );
  }
  return parsed;
}
```

Now update each step in `src/workflow.ts` to validate:

```ts
// Replace the requirements step in src/workflow.ts
const requirements = await step.run("extract-requirements", async () => {
  const raw = await callAgent<RequirementsOutput>(REQUIREMENTS_PROMPT, brief);
  return validateKeys(raw, ["functional", "nonFunctional", "constraints", "actors"]);
});

const architecture = await step.run("design-architecture", async () => {
  const raw = await callAgent<ArchitectureOutput>(
    ARCHITECTURE_PROMPT,
    JSON.stringify(requirements)
  );
  return validateKeys(raw, ["components", "dataFlow", "techChoices"]);
});

const apiContract = await step.run("define-api-contract", async () => {
  const raw = await callAgent<APIContractOutput>(
    API_CONTRACT_PROMPT,
    JSON.stringify(architecture)
  );
  return validateKeys(raw, ["endpoints"]);
});

const testPlan = await step.run("write-test-plan", async () => {
  const raw = await callAgent<TestPlanOutput>(
    TEST_PLAN_PROMPT,
    JSON.stringify(apiContract)
  );
  return validateKeys(raw, ["cases"]);
});
```

Import `validateKeys` at the top of `workflow.ts`:

```ts
import { callAgent, validateKeys } from "./callAgent";
```

## Fixed state

```text
DIAGRAM: Pipeline with typed validation at every handoff
Caption: Each step validates output shape before the next agent receives it
Nodes:
1. Orchestrator Function - step-isolated runner
2. Agent 1 (Requirements) - output validated for functional, nonFunctional, constraints, actors
3. Agent 2 (Architecture) - output validated for components, dataFlow, techChoices
4. Agent 3 (API Contract) - output validated for endpoints
5. Agent 4 (Test Plan) - output validated for cases
6. Validation gate - JSON.parse plus required key check at every boundary
Flow:
- Brief enters orchestrator
- Agent 1 produces JSON, validation gate checks shape, passes to Agent 2
- Agent 2 produces JSON, validation gate checks shape, passes to Agent 3
- Agent 3 produces JSON, validation gate checks shape, passes to Agent 4
- Agent 4 produces JSON, validation gate checks shape, result returned
- Any validation failure triggers a retry on that step only
```

## After

You send a product brief as a single event. Sixty seconds later, you have structured requirements, architecture decisions, an API contract with typed endpoints, and a test plan with specific assertions. Each intermediate result sits in the dashboard as inspectable JSON. When the test plan looks wrong, you click into step three and see the API contract that caused it. You fix the API contract prompt, re-run one step, and the downstream output updates. No meeting. No Slack thread. No action items. Four agents ran a clean relay race and each one passed a baton the next one could actually hold.

## Takeaway

The pattern is typed boundaries between agents. Agents are not team members who can ask clarifying questions. They are functions. Functions need type signatures. When you define the exact shape of data that crosses each boundary, you can validate it, retry it, and inspect it independently. Apply this to any multi-agent chain: define the handoff types first, then write the prompts.