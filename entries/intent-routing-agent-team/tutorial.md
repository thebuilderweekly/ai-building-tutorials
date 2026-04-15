## Opening thesis

You will build a router agent that classifies incoming customer queries into one of five categories and dispatches each query to a specialist agent trained on that category alone. The router does one thing: classify. The specialists do one thing each: billing, technical support, product questions, account management, or refunds. A router that dispatches to five specialists handles a thousand customer queries per hour with consistent expertise per category, which a single generalist agent handles slower and worse. By the end, every query reaches an agent that knows exactly that domain and nothing else.

## Before

Your support agent runs on one prompt. The prompt tells Claude it is a customer support agent and gives it a list of common topics. A customer asks about a billing dispute. The agent responds with general empathy and a vague mention of the refund policy. The customer asks again with more detail. The agent repeats the empathy. The customer asks for a manager. A different customer asks why their API integration returns a 502 error. The agent suggests checking their internet connection. The customer asks for a manager. Every escalation costs you a real person's time. Every generalist response erodes trust. The agent is fast but consistently mediocre, because no single prompt can hold deep expertise across billing, infrastructure, product roadmap, account permissions, and refund policy at once.

## Architecture

The system has one router agent and five specialist agents. The router reads each incoming query and returns a single category label plus a confidence score. If confidence is high, the query routes to the matching specialist. If confidence is low, the query routes to a fallback path that asks the customer to clarify. Each specialist has its own system prompt, its own tool list, and its own response style. Specialists never talk to each other. The router never answers questions itself.

```text
DIAGRAM: Intent routing agent team
Caption: One router classifies queries and dispatches to five specialists with no overlap
Nodes:
1. Customer query - The incoming message from the user
2. Router agent (Claude) - Classifies into one of five categories with confidence
3. Confidence gate - Routes high-confidence queries to specialists, low-confidence to clarification
4. Billing specialist - Handles refunds, invoices, payment failures
5. Technical specialist - Handles API errors, integration issues, debugging
6. Product specialist - Handles feature questions, roadmap, capabilities
7. Account specialist - Handles permissions, team members, plan changes
8. Refunds specialist - Handles refund requests, policy questions, dispute resolution
9. Clarification path - Asks the customer to restate when intent is unclear
10. Response - Specialist response sent back to the customer
Flow:
- Customer query reaches Router agent
- Router agent returns category and confidence score
- Confidence gate checks if confidence exceeds threshold
- If high: gate dispatches query to matching specialist
- If low: gate sends query to Clarification path
- Specialist generates response using its own system prompt and tools
- Response goes back to Customer
```

## Step-by-step implementation

### Step 1: Install dependencies

You need the Anthropic SDK. Get your API key from https://console.anthropic.com/settings/keys.

```bash
pip install anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Step 2: Define the categories and confidence threshold

Categories must be mutually exclusive and exhaustive. Five is a good starting number: enough to specialize meaningfully, few enough that the router rarely guesses wrong. Save this as `categories.py`.

```python
# categories.py

CATEGORIES = ["billing", "technical", "product", "account", "refunds"]

CONFIDENCE_THRESHOLD = 0.75  # below this, route to clarification

CATEGORY_DESCRIPTIONS = {
    "billing": "Invoices, payment methods, payment failures, charges the customer does not recognize, billing schedule questions",
    "technical": "API errors, integration bugs, performance issues, error codes, debugging help, environment setup",
    "product": "What features exist, how features work, roadmap, capabilities, comparing tiers, integration possibilities",
    "account": "User permissions, team members, plan upgrades, plan downgrades, transferring ownership, deleting account",
    "refunds": "Requesting a refund, refund eligibility, refund status, disputing a charge, cancellation with refund",
}
```

### Step 3: Build the router

The router has one job: read the query and return a category label plus confidence. The system prompt lists every category with its scope. The router returns structured JSON. It never tries to answer the query itself.

```python
# router.py
import os
import json
import anthropic
from categories import CATEGORIES, CATEGORY_DESCRIPTIONS

client = anthropic.Anthropic()

def build_router_prompt():
    category_block = "\n".join(
        f"- {cat}: {desc}" for cat, desc in CATEGORY_DESCRIPTIONS.items()
    )
    return f"""You are a query router. Your only job is to classify incoming customer queries into one of five categories.

Categories:
{category_block}

Rules:
- Return EXACTLY one category from the list above.
- Return a confidence score between 0.0 and 1.0.
- Confidence reflects how clearly the query matches the chosen category.
- If a query could fit two categories, pick the more specific one and lower the confidence.
- Do NOT attempt to answer the query. You only classify.

Return ONLY a JSON object: {{"category": "...", "confidence": 0.XX, "reason": "one short sentence"}}"""

ROUTER_PROMPT = build_router_prompt()

def route(query: str) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=200,
        system=ROUTER_PROMPT,
        messages=[{"role": "user", "content": query}],
    )
    text = response.content[0].text.strip()
    if text.startswith("```"):
        text = text.split("\n", 1)[1].rsplit("```", 1)[0].strip()
    return json.loads(text)
```

### Step 4: Build a specialist factory

Every specialist has the same shape: a system prompt, a tool list, and a response function. Rather than writing five nearly-identical files, use a factory pattern. Each specialist is constructed from a config dict.

```python
# specialist.py
import anthropic

client = anthropic.Anthropic()

class Specialist:
    def __init__(self, name: str, system_prompt: str, tools: list = None, max_tokens: int = 1024):
        self.name = name
        self.system_prompt = system_prompt
        self.tools = tools or []
        self.max_tokens = max_tokens

    def respond(self, query: str, context: dict = None) -> str:
        messages = [{"role": "user", "content": query}]
        kwargs = {
            "model": "claude-sonnet-4-20250514",
            "max_tokens": self.max_tokens,
            "system": self.system_prompt,
            "messages": messages,
        }
        if self.tools:
            kwargs["tools"] = self.tools
        response = client.messages.create(**kwargs)
        return response.content[0].text
```

### Step 5: Configure each specialist

Each specialist gets a system prompt scoped to its category. The prompts are short and concrete. They do not try to handle out-of-scope queries gracefully; if a query reaches the wrong specialist, the specialist should escalate rather than answer.

```python
# specialists_config.py
from specialist import Specialist

BILLING_PROMPT = """You are a billing specialist. You answer questions about invoices, payment methods, payment failures, and unrecognized charges.

Rules:
- Always confirm the customer's account email before discussing specific charges.
- Quote exact dates and amounts when referencing invoices.
- For disputed charges, confirm the charge details first, then explain the resolution path.
- If the question is not about billing, respond: "This question seems to be about [topic]. Let me route you to the right team."
- Be concise. Customers asking about money want clarity, not empathy theater."""

TECHNICAL_PROMPT = """You are a technical support specialist. You answer questions about API errors, integration bugs, and debugging.

Rules:
- Always ask for the exact error message and the request that caused it.
- Suggest the most likely root cause based on the error code.
- Provide a code example showing the correct usage if relevant.
- For 5xx errors, check the status page link before assuming customer code is at fault.
- If the question is not about technical issues, respond: "This question seems to be about [topic]. Let me route you to the right team."
"""

PRODUCT_PROMPT = """You are a product specialist. You answer questions about features, capabilities, and what the product can or cannot do.

Rules:
- Be specific about which plan tier includes a feature.
- Distinguish between features that exist, features on the roadmap, and features not planned.
- Never promise features that are not confirmed in the public roadmap.
- For comparison questions, list facts side by side.
- If the question is not about product features, respond: "This question seems to be about [topic]. Let me route you to the right team."
"""

ACCOUNT_PROMPT = """You are an account management specialist. You answer questions about permissions, team members, and plan changes.

Rules:
- For destructive actions (downgrades, deletions, ownership transfers), confirm intent before proceeding.
- For permission questions, list the exact roles and what each can do.
- For plan changes, surface any data export or feature loss the customer should know about.
- If the question is not about account management, respond: "This question seems to be about [topic]. Let me route you to the right team."
"""

REFUNDS_PROMPT = """You are a refunds specialist. You handle refund requests, eligibility questions, and disputes.

Rules:
- Confirm the customer's account and the specific charge being disputed.
- State the refund eligibility window clearly.
- For approved refunds, state the timeline (3-5 business days for card, 7-10 for ACH).
- For denied refunds, explain why and offer the next escalation path.
- If the question is not about refunds, respond: "This question seems to be about [topic]. Let me route you to the right team."
"""

SPECIALISTS = {
    "billing": Specialist("billing", BILLING_PROMPT),
    "technical": Specialist("technical", TECHNICAL_PROMPT),
    "product": Specialist("product", PRODUCT_PROMPT),
    "account": Specialist("account", ACCOUNT_PROMPT),
    "refunds": Specialist("refunds", REFUNDS_PROMPT),
}
```

### Step 6: Build the orchestrator

The orchestrator ties the router and specialists together. It calls the router, checks confidence, and either dispatches to a specialist or asks the customer to clarify.

```python
# orchestrator.py
from router import route
from specialists_config import SPECIALISTS
from categories import CONFIDENCE_THRESHOLD

CLARIFICATION_RESPONSE = "I want to make sure I get you to the right person. Could you tell me a bit more? Are you asking about your bill, a technical issue, how a feature works, your account settings, or a refund?"

def handle_query(query: str) -> dict:
    routing = route(query)
    category = routing["category"]
    confidence = routing["confidence"]

    if confidence < CONFIDENCE_THRESHOLD:
        return {
            "category": "clarification",
            "confidence": confidence,
            "response": CLARIFICATION_RESPONSE,
            "router_reason": routing.get("reason", ""),
        }

    specialist = SPECIALISTS.get(category)
    if specialist is None:
        return {
            "category": "unknown",
            "confidence": confidence,
            "response": CLARIFICATION_RESPONSE,
            "router_reason": f"Router returned unknown category: {category}",
        }

    response = specialist.respond(query)
    return {
        "category": category,
        "confidence": confidence,
        "response": response,
        "router_reason": routing.get("reason", ""),
    }

if __name__ == "__main__":
    queries = [
        "Why was I charged twice on March 14?",
        "My API calls are returning 502 errors since this morning, what's happening?",
        "Does the Pro plan include team SSO?",
        "How do I remove a team member who left the company?",
        "I want a refund for last month, I forgot to cancel before the renewal date.",
        "Hi, I have a question.",
    ]
    for q in queries:
        result = handle_query(q)
        print(f"\nQuery: {q}")
        print(f"  Routed to: {result['category']} (confidence {result['confidence']})")
        print(f"  Response: {result['response'][:200]}...")
```

### Step 7: Add logging for routing decisions

Every routing decision should log the query, the chosen category, the confidence, and the router's reason. This lets you audit misroutes and tune category descriptions over time.

```python
# logger.py
import json
import os
from datetime import datetime

LOG_PATH = os.environ.get("ROUTING_LOG_PATH", "routing.jsonl")

def log_routing(query: str, result: dict):
    entry = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "query": query,
        "category": result["category"],
        "confidence": result["confidence"],
        "router_reason": result.get("router_reason", ""),
    }
    with open(LOG_PATH, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

Update `orchestrator.py` to call `log_routing` after every query.

### Step 8: Review misroutes and tune categories

Every week, read the routing log and look for clarification entries with high confidence (the router was sure but still flagged) or specialist responses that quoted the "this question seems to be about" escalation. Both indicate category boundaries that need tightening.

```bash
# Find queries the router was confident about but still hit clarification
cat routing.jsonl | python3 -c "
import json, sys
for line in sys.stdin:
    e = json.loads(line)
    if e['category'] == 'clarification' and e['confidence'] > 0.6:
        print(f\"Confident-but-clarified: {e['query']} (confidence {e['confidence']})\")
"
```

When you find a pattern (e.g., "billing" misclassified as "refunds"), edit the category description in `categories.py` to disambiguate, and rerun the test suite.

## Breakage

Skip the router. Use one generalist prompt that lists all five domains. The agent now has to hold five domains of expertise in one context, prioritize them based on the query, and choose a tone. It does this poorly. Billing questions get product-marketing flavored answers because the prompt mentions both. Technical questions get refund policy disclaimers because the prompt covers refunds too. Every response is a mediocre blend of every specialist's style. The customer cannot tell what kind of expert they are talking to. Trust drops. Escalations climb. The agent is fast and consistently unhelpful.

```text
DIAGRAM: Generalist agent failure mode
Caption: One agent trying to handle five domains produces blended, low-expertise responses
Nodes:
1. Customer query - Any kind of incoming question
2. Generalist agent (Claude) - Single prompt covering all five domains
3. Blended response - Mixes tone and expertise across domains
4. Customer - Receives unfocused answer, escalates to human
Flow:
- Customer query reaches Generalist agent
- Generalist agent processes query against a five-domain prompt
- Response blends terminology and tone from multiple domains
- Customer cannot identify the expert speaking to them
- Customer escalates to a human for a clear answer
```

## The fix

The fix is the router from Step 3 plus the specialist factory from Step 4. The router does classification only and never answers. The specialists answer only within their category and explicitly escalate if a query is out of scope. The critical mechanism is the confidence threshold from Step 6, isolated below. Without this gate, low-confidence routes would silently dispatch to the wrong specialist, and you would learn about misroutes through escalations instead of through the log.

```python
# The confidence gate from orchestrator.py, isolated
def handle_query(query: str) -> dict:
    routing = route(query)
    category = routing["category"]
    confidence = routing["confidence"]

    # The gate: if the router is not confident, do not guess.
    # Ask the customer to clarify before dispatching to any specialist.
    if confidence < CONFIDENCE_THRESHOLD:
        return {
            "category": "clarification",
            "confidence": confidence,
            "response": CLARIFICATION_RESPONSE,
        }

    # Only dispatch when confidence is high.
    specialist = SPECIALISTS.get(category)
    response = specialist.respond(query)
    return {"category": category, "confidence": confidence, "response": response}
```

The gate turns a routing system from "always make a choice" into "make a choice only when sure." Misroutes stop reaching customers. The clarification path catches ambiguous queries before they cause damage.

## Fixed state

```text
DIAGRAM: Routed agent team with confidence gate
Caption: Router classifies, gate filters by confidence, specialists answer only in their domain
Nodes:
1. Customer query - The incoming message
2. Router agent - Returns category and confidence
3. Confidence gate - Threshold check
4. Clarification path - Active when confidence is below threshold
5. Billing specialist - Handles billing queries with billing-specific prompt
6. Technical specialist - Handles technical queries with technical-specific prompt
7. Product specialist - Handles product queries with product-specific prompt
8. Account specialist - Handles account queries with account-specific prompt
9. Refunds specialist - Handles refund queries with refund-specific prompt
10. Routing log - Records every routing decision for audit and tuning
11. Customer response - Specialist or clarification response
Flow:
- Customer query reaches Router agent
- Router agent returns category and confidence
- Confidence gate evaluates threshold
- If below threshold, Clarification path responds
- If above threshold, query routes to matching specialist
- Specialist generates domain-expert response
- Routing decision logs to Routing log
- Customer receives the response
```

## After

A customer asks about a double charge. The router returns "billing" with confidence 0.94. The billing specialist asks for the account email, confirms the charge, identifies the duplicate, and processes the reversal. Total exchange: three messages, no escalation. A customer asks about a 502 error. The router returns "technical" with confidence 0.91. The technical specialist asks for the request payload, identifies a malformed header, and provides a corrected code example. Total exchange: two messages, no escalation. A customer says "Hi, I have a question." The router returns "billing" with confidence 0.34. The clarification path responds, asking the customer to specify. The customer clarifies, the router classifies with high confidence, and the right specialist takes over. Your weekly report shows escalation rate down from 28% to 6%. Average resolution time down from 14 minutes to 3. Customer satisfaction up across every category, because every customer is talking to an expert in exactly the thing they need.

## Takeaway

The pattern is router plus specialists with a confidence gate. The router holds no expertise; it holds only the taxonomy. The specialists hold deep expertise in narrow domains. The gate refuses to dispatch when the router is uncertain, preventing silent misroutes. Apply this anywhere you have an LLM that needs to handle a wide range of inputs with consistent quality. The generalist tax is real: a single prompt covering many domains will always underperform a router-plus-specialists architecture, because no prompt can hold deep expertise in every domain at once.
