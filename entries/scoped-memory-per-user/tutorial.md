## Opening thesis

You will build a Python agent that writes and reads memory scoped to individual user IDs using mem0. No shared state. No cross-user leakage. An agent can index and retrieve against millions of per-user memory stores in parallel, which a human operating the same product would need a team and a custom dashboard to match. By the end, every memory operation your agent performs is namespaced, and retrieval only returns the requesting user's own history.

## Before

You have one memory store. Every user's conversation history, preferences, and context sit in the same bucket. User A asks about their dietary restrictions, and the agent recalls User B's nut allergy. Your support agent confidently tells a vegan customer about the steak recommendations it memorized from someone else's session last Tuesday. You know the fix is "just namespace it," but your current setup has no isolation layer. You are one angry customer away from a privacy incident. The manual alternative: a human operator builds a per-user lookup table, maintains a dashboard to inspect and prune entries, and coordinates a small team to handle the load. That does not scale past a few hundred users.

## Architecture

The system has three components: your application server, the mem0 client, and mem0's hosted memory API. The application server receives a user request, extracts the user ID, and passes it to every mem0 call. mem0 handles vector storage, indexing, and retrieval. The user ID acts as the partition key. No query ever crosses that boundary.

```text
DIAGRAM: Per-user memory architecture
Caption: Shows how user requests flow through the app server into namespaced mem0 stores.
Nodes:
1. User (browser/client) - sends requests with a session token
2. App Server (Python) - extracts user_id, calls mem0 client
3. mem0 Client (SDK) - adds user_id to every add/search call
4. mem0 API (hosted) - stores and retrieves vectors partitioned by user_id
Flow:
- User sends message to App Server
- App Server extracts user_id from session
- App Server calls mem0 Client with user_id and message
- mem0 Client sends namespaced request to mem0 API
- mem0 API returns only memories matching that user_id
- App Server returns response to User
```

## Step-by-step implementation

### 1. Install mem0

Install the mem0 Python SDK. This is the only dependency beyond the standard library.

```bash
pip install mem0ai
```

### 2. Set your API key

Get your mem0 API key from the [mem0 dashboard](https://app.mem0.ai/). Set it as an environment variable. Every code block below reads from this variable.

```bash
export MEM0_API_KEY="m0-your-actual-key-here"
```

### 3. Initialize the mem0 client

Create a file called `agent.py`. Initialize the mem0 client using the API key from the environment. This client handles all communication with the mem0 API.

```python
import os
from mem0 import MemoryClient

client = MemoryClient(api_key=os.environ["MEM0_API_KEY"])
```

### 4. Write a function to add scoped memories

Every call to `client.add` accepts a `user_id` parameter. This parameter is the namespace. Memories added under one user ID are invisible to queries under a different user ID. The function below takes a user ID, a list of messages, and stores them.

```python
def add_memory(user_id: str, messages: list[dict]) -> dict:
    result = client.add(messages, user_id=user_id)
    return result
```

### 5. Write a function to retrieve scoped memories

Retrieval also takes a `user_id`. The `client.search` method returns only memories that belong to that user. The `query` parameter is the natural language search string.

```python
def search_memory(user_id: str, query: str) -> list:
    results = client.search(query, user_id=user_id)
    return results
```

### 6. Add memories for two separate users

Simulate two users. User `u_alice` tells the agent she is vegetarian. User `u_bob` tells the agent he prefers window seats. These two facts must never cross.

```python
add_memory(
    user_id="u_alice",
    messages=[
        {"role": "user", "content": "I am vegetarian. Please remember that."},
        {"role": "assistant", "content": "Got it. I will remember you are vegetarian."}
    ]
)

add_memory(
    user_id="u_bob",
    messages=[
        {"role": "user", "content": "I always want a window seat on flights."},
        {"role": "assistant", "content": "Noted. Window seat preference saved."}
    ]
)

print("Memories stored for both users.")
```

### 7. Query each user's memory and verify isolation

Search for "food preferences" under both user IDs. Alice should see her vegetarian preference. Bob should see nothing related to food. This is the isolation test.

```python
alice_results = search_memory(user_id="u_alice", query="food preferences")
bob_results = search_memory(user_id="u_bob", query="food preferences")

print("Alice's food results:")
for mem in alice_results:
    print(f"  {mem['memory']}")

print("Bob's food results:")
for mem in bob_results:
    print(f"  {mem['memory']}")
```

### 8. List all memories for a single user

The `client.get_all` method with a `user_id` returns every memory stored under that namespace. Use this for debugging or building a user-facing "your data" page.

```python
def list_all_memories(user_id: str) -> list:
    memories = client.get_all(user_id=user_id)
    return memories

alice_all = list_all_memories("u_alice")
print(f"Alice has {len(alice_all)} memories stored.")
for mem in alice_all:
    print(f"  [{mem['id']}] {mem['memory']}")
```

### 9. Delete a specific user's memory

When a user requests data deletion, you remove their memories without touching anyone else's store. The `client.delete_all` method with a `user_id` wipes that namespace.

```python
def delete_user_memories(user_id: str) -> None:
    client.delete_all(user_id=user_id)
    print(f"All memories deleted for {user_id}.")

delete_user_memories("u_alice")

# Verify Alice's memories are gone
alice_remaining = list_all_memories("u_alice")
print(f"Alice now has {len(alice_remaining)} memories.")

# Verify Bob's memories are untouched
bob_remaining = list_all_memories("u_bob")
print(f"Bob still has {len(bob_remaining)} memories.")
```

### 10. Run the full script

Execute the complete file. You should see Alice's vegetarian memory appear only for Alice, Bob's window seat only for Bob, and Alice's deletion leave Bob untouched.

```bash
python agent.py
```

## Breakage

Remove the `user_id` parameter from every `client.add` and `client.search` call. Now every memory goes into a global namespace. When Bob searches for "food preferences," he gets Alice's vegetarian preference. When Alice searches for "seat preferences," she gets Bob's window seat. In a production system with thousands of users, this is not a hypothetical. It is a data leak. The agent confidently serves one user's private context to another. The trust model collapses.

```text
DIAGRAM: Breakage, no user_id namespace
Caption: Without user_id, all memories land in a single global store and leak across users.
Nodes:
1. User Alice - sends food preference
2. User Bob - sends seat preference
3. App Server - calls mem0 without user_id
4. mem0 API (global store) - single namespace for all users
Flow:
- Alice sends "I am vegetarian" to App Server
- App Server calls client.add(messages) with NO user_id
- mem0 API stores memory in global namespace
- Bob sends "food preferences" query to App Server
- App Server calls client.search(query) with NO user_id
- mem0 API returns Alice's vegetarian memory to Bob
- Bob sees Alice's private data
```

## The fix

The fix is exactly what Steps 4 and 5 already show: pass `user_id` on every call. But the real safeguard is a wrapper that makes it impossible to forget. Create a class that requires a user ID at construction time. Every method on the class passes that ID automatically. No engineer on your team can accidentally omit it.

```python
class ScopedMemory:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.client = MemoryClient(api_key=os.environ["MEM0_API_KEY"])

    def add(self, messages: list[dict]) -> dict:
        return self.client.add(messages, user_id=self.user_id)

    def search(self, query: str) -> list:
        return self.client.search(query, user_id=self.user_id)

    def get_all(self) -> list:
        return self.client.get_all(user_id=self.user_id)

    def delete_all(self) -> None:
        self.client.delete_all(user_id=self.user_id)


# Usage: impossible to forget the user_id
alice_mem = ScopedMemory("u_alice")
alice_mem.add([
    {"role": "user", "content": "I am vegetarian."},
    {"role": "assistant", "content": "Noted."}
])
results = alice_mem.search("food preferences")
print(results)
```

## Fixed state

```text
DIAGRAM: Fixed architecture with ScopedMemory wrapper
Caption: The ScopedMemory class enforces user_id on every mem0 call, preventing cross-user leakage.
Nodes:
1. User Alice - sends request
2. User Bob - sends request
3. App Server - creates ScopedMemory(user_id) per request
4. ScopedMemory("u_alice") - wraps mem0 client with Alice's ID
5. ScopedMemory("u_bob") - wraps mem0 client with Bob's ID
6. mem0 API - stores and retrieves vectors partitioned by user_id
Flow:
- Alice request hits App Server
- App Server creates ScopedMemory("u_alice")
- ScopedMemory("u_alice") calls mem0 API with user_id="u_alice"
- mem0 API returns only Alice's memories
- Bob request hits App Server
- App Server creates ScopedMemory("u_bob")
- ScopedMemory("u_bob") calls mem0 API with user_id="u_bob"
- mem0 API returns only Bob's memories
- No cross-user data is ever returned
```

## After

You have one memory store, but it behaves like millions. Every user's context lives in its own namespace. Alice searches and sees only her history. Bob searches and sees only his. Your `ScopedMemory` wrapper enforces the partition at the code level, so a missed parameter cannot cause a leak. Deletion is per-user and instant. No dashboard. No team. No manual coordination. The agent handles the indexing, retrieval, and isolation across every user in parallel. A human doing the same work would need a dedicated team, a custom admin interface, and careful access controls for each tenant. The agent does it with a constructor argument.

## Takeaway

The pattern is: make the correct behavior the only path. Wrap your memory client so that the namespace parameter is required at initialization, not optional at call time. Apply this to any multi-tenant resource: vector stores, caches, file systems, session data. If isolation depends on a developer remembering a keyword argument, isolation will fail. Make the wrapper enforce it.
