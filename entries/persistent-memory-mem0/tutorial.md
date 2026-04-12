## Opening thesis

You will build a Python agent that stores facts from every conversation into a persistent memory layer, then retrieves them before answering new questions. An agent with persistent memory improves its responses over time without retraining, which a human doing the same job would need a notebook and perfect recall to match. The tool that makes this possible is [mem0](https://docs.mem0.ai/).

## Before

Your agent forgets everything when the session ends. A user tells it they prefer metric units on Monday. On Tuesday, it asks again. A user explains their project architecture in detail. Next session, the agent has no idea the project exists. You have two options: stuff the entire chat history into the context window (expensive, slow, hits token limits fast) or accept that every conversation starts from zero. A human support agent with a notebook would outperform your bot after a week, because the human remembers things. Your agent does not.

## Architecture

The system has three components: a Python agent script, the mem0 memory service, and an LLM provider (Anthropic Claude). When a user sends a message, the agent queries mem0 for relevant memories before calling the LLM. After the LLM responds, the agent stores new facts from the conversation back into mem0. This loop means the agent accumulates knowledge across restarts, sessions, and even machines.

```text
DIAGRAM: Persistent Memory Agent Loop
Caption: How the agent reads and writes memory on every turn.
Nodes:
1. User Input - the message from the user
2. mem0 Search - retrieves relevant memories for this user
3. Prompt Builder - combines memories + user input into a prompt
4. Anthropic Claude - generates a response given the enriched prompt
5. Response Output - the answer sent back to the user
6. mem0 Add - extracts and stores new facts from the exchange
Flow:
- User Input sends the message to mem0 Search and Prompt Builder
- mem0 Search returns relevant memories to Prompt Builder
- Prompt Builder sends enriched prompt to Anthropic Claude
- Anthropic Claude returns response to Response Output
- User Input + Response Output are sent to mem0 Add for storage
```

## Step-by-step implementation

### 1. Install dependencies

You need two packages: the mem0 client and the Anthropic SDK. Both install from PyPI.

```bash
pip install mem0ai anthropic
```

### 2. Set your API keys

You need a mem0 API key and an Anthropic API key. Get the mem0 key from https://app.mem0.ai/dashboard/api-keys. Get the Anthropic key from https://console.anthropic.com/settings/keys. Export both as environment variables.

```bash
export MEM0_API_KEY="your_mem0_api_key_here"
export ANTHROPIC_API_KEY="your_anthropic_api_key_here"
```

### 3. Initialize the clients

Create a file called `agent.py`. Import the libraries and initialize both clients. The mem0 client uses your API key to connect to the hosted memory service.

```python
# agent.py
import os
from mem0 import MemoryClient
import anthropic

mem0_client = MemoryClient(api_key=os.environ["MEM0_API_KEY"])
llm_client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

USER_ID = "user_42"
```

### 4. Build the memory retrieval function

Before every response, the agent searches mem0 for memories related to the current message. The `search` method returns a list of memory objects ranked by relevance. Each object has a `memory` field containing the stored text.

```python
def retrieve_memories(query: str, user_id: str) -> list[str]:
    results = mem0_client.search(query, user_id=user_id, limit=5)
    memories = [r["memory"] for r in results if "memory" in r]
    return memories
```

### 5. Build the prompt with memory context

This function takes the user message and the retrieved memories, then formats them into a system prompt and user message. The system prompt tells the LLM it has access to prior knowledge about this user.

```python
def build_prompt(user_message: str, memories: list[str]) -> tuple[str, str]:
    if memories:
        memory_block = "\n".join(f"- {m}" for m in memories)
        system = (
            "You are a helpful assistant. You have the following memories "
            "about this user from previous conversations:\n"
            f"{memory_block}\n\n"
            "Use these memories to personalize your response. "
            "Do not mention that you have a memory system unless asked."
        )
    else:
        system = "You are a helpful assistant."
    return system, user_message
```

### 6. Call the LLM

Send the enriched prompt to Claude. This is a standard Anthropic API call. The model receives both the system prompt (with memories) and the user message.

```python
def call_llm(system: str, user_message: str) -> str:
    response = llm_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system=system,
        messages=[{"role": "user", "content": user_message}],
    )
    return response.content[0].text
```

### 7. Store new memories after each exchange

After the agent responds, store the conversation turn in mem0. The `add` method extracts facts automatically and stores them. You pass both the user message and the assistant response so mem0 can extract relevant details from the full exchange.

```python
def store_memories(user_message: str, assistant_response: str, user_id: str):
    messages = [
        {"role": "user", "content": user_message},
        {"role": "assistant", "content": assistant_response},
    ]
    mem0_client.add(messages, user_id=user_id)
```

### 8. Wire it all together in a conversation loop

This is the main loop. It reads user input, retrieves memories, builds the prompt, calls the LLM, prints the response, and stores new memories. When you kill the process and restart it, the memories persist. That is the entire point.

```python
def main():
    print("Agent ready. Type 'quit' to exit.")
    while True:
        user_input = input("\nYou: ").strip()
        if user_input.lower() == "quit":
            break

        memories = retrieve_memories(user_input, USER_ID)
        if memories:
            print(f"  [Retrieved {len(memories)} memories]")

        system, message = build_prompt(user_input, memories)
        response = call_llm(system, message)
        print(f"\nAgent: {response}")

        store_memories(user_input, response, USER_ID)
        print("  [Memories updated]")


if __name__ == "__main__":
    main()
```

### 9. Run it and verify persistence

Start a session, tell the agent something specific, then quit and restart. The agent should remember what you said.

```bash
python agent.py
# Session 1:
# You: I prefer all temperatures in Celsius, never Fahrenheit.
# Agent: Got it, I'll use Celsius for any temperature references.
# You: quit

python agent.py
# Session 2:
# You: What's a comfortable room temperature?
# Agent: Around 21 to 22 degrees Celsius is comfortable for most people.
# (Note: it used Celsius without being asked, because it remembered.)
```

## Breakage

If you skip the memory storage step (step 7), the agent works fine within a single session but loses all context on restart. The LLM still responds. The code still runs. But the agent is stateless. Every conversation is isolated. Over ten sessions, the agent learns nothing. A human colleague doing the same job, even one with average memory, would dramatically outperform it by session three because they remember that the user prefers Celsius, works on a Django project, and hates verbose explanations.

```text
DIAGRAM: Broken Agent Without Memory Storage
Caption: Without the store step, the agent never accumulates knowledge.
Nodes:
1. User Input - the message from the user
2. mem0 Search - returns empty results every time
3. Prompt Builder - builds a generic prompt with no memories
4. Anthropic Claude - generates a generic response
5. Response Output - the answer, with no personalization
6. mem0 Add - MISSING, never called
Flow:
- User Input sends the message to mem0 Search and Prompt Builder
- mem0 Search returns nothing (no memories were ever stored)
- Prompt Builder sends generic prompt to Anthropic Claude
- Anthropic Claude returns generic response to Response Output
- No data flows to mem0 Add because the step does not exist
```

## The fix

The fix is the `store_memories` call at the end of each turn. You already wrote this function in step 7. The critical line is in the `main()` loop: `store_memories(user_input, response, USER_ID)`. If you removed it during debugging or forgot to include it, add it back. Here is the corrected loop with the storage call highlighted by a comment.

```python
def main():
    print("Agent ready. Type 'quit' to exit.")
    while True:
        user_input = input("\nYou: ").strip()
        if user_input.lower() == "quit":
            break

        memories = retrieve_memories(user_input, USER_ID)
        if memories:
            print(f"  [Retrieved {len(memories)} memories]")

        system, message = build_prompt(user_input, memories)
        response = call_llm(system, message)
        print(f"\nAgent: {response}")

        # THIS IS THE FIX: store memories after every exchange
        store_memories(user_input, response, USER_ID)
        print("  [Memories updated]")


if __name__ == "__main__":
    main()
```

## Fixed state

```text
DIAGRAM: Agent With Persistent Memory Active
Caption: The full loop with memory storage and retrieval working.
Nodes:
1. User Input - the message from the user
2. mem0 Search - retrieves relevant memories for this user
3. Prompt Builder - combines memories + user input into enriched prompt
4. Anthropic Claude - generates a personalized response
5. Response Output - the personalized answer
6. mem0 Add - extracts and stores new facts from the exchange
Flow:
- User Input sends the message to mem0 Search and Prompt Builder
- mem0 Search returns relevant memories to Prompt Builder
- Prompt Builder sends enriched prompt to Anthropic Claude
- Anthropic Claude returns personalized response to Response Output
- User Input + Response Output are sent to mem0 Add
- mem0 Add stores new facts, available for future mem0 Search calls
```

## After

Your agent remembers everything across restarts. A user mentions they prefer Celsius. Next session, the agent uses Celsius without asking. A user describes their tech stack. A week later, the agent references it correctly. You did not retrain anything. You did not increase the context window. You added a read step before the prompt and a write step after the response. The agent now accumulates knowledge the way a human with perfect notes would, except it never misfiles anything, never forgets to check its notes, and never takes a sick day. Over ten sessions, it compounds. Over a hundred, it knows things about the user that would fill pages. A human doing the same job would need a notebook and perfect recall to match.

## Takeaway

The pattern is: read before you respond, write after you respond. This applies to any agent, any memory backend, any LLM. Persistent memory is not a feature you add at the end. It is the difference between a stateless function and an agent that gets better every time it runs.