## Opening thesis

You will build a retrieval pipeline that combines Qdrant vector search with metadata filters and an Anthropic-powered agent to pull exactly one correct chunk from a collection of ten million. An agent combining semantic search with metadata filters reaches into ten million chunks and returns the right one in under a hundred milliseconds, which a human searching a SQL database by hand cannot come close to matching. The secret is not better embeddings. It is structured metadata applied at query time.

## Before

You have a RAG agent. A user asks: "What was my invoice total for March 2026?" Your agent embeds that query, fires it at a vector store, and gets back ten chunks. Three are invoices from other users. Two are from the right user but wrong months. One is a refund notice that mentions March. Four are thematically adjacent noise about billing policy. You stare at the results and realize the embedding did its job: all ten chunks are semantically close to "invoice total March 2026." The embedding cannot know that `user_id` matters. It cannot know that `document_date` matters. So your agent hallucinates an answer from the wrong chunk, and the user gets a number that belongs to someone else. You catch it in testing. You add a reranker. The reranker picks a slightly better wrong chunk. The problem is not ranking. The problem is that pure semantic search has no concept of access control or temporal scope.

## Architecture

The system has four components. Qdrant stores vectors with payload metadata. The Anthropic API generates embeddings via `voyage-3` (accessible through the Anthropic ecosystem) and handles the final answer generation via Claude. A Python orchestrator wires them together. Metadata filters on `user_id` and `document_date` narrow the search space before the vector similarity computation runs.

```text
DIAGRAM: Filtered vector retrieval pipeline
Caption: Query flows from user through metadata filter construction to Qdrant filtered search, then to Claude for answer generation.
Nodes:
1. User query - natural language question with implicit user and date context
2. Orchestrator (Python) - constructs metadata filters and embedding, coordinates calls
3. Anthropic Embedding API - converts query text to a vector via voyage-3
4. Qdrant - stores 10M chunks with payload fields: user_id, document_date, doc_type
5. Anthropic Claude API - receives the retrieved chunk and generates the final answer
Flow:
- User query enters the Orchestrator
- Orchestrator sends query text to Anthropic Embedding API, receives vector
- Orchestrator builds a Qdrant filter: must match user_id AND document_date range
- Orchestrator sends vector + filter to Qdrant, receives top-1 chunk
- Orchestrator sends chunk + original query to Claude API
- Claude API returns the final answer to the user
```

## Step-by-step implementation

### Step 1: Install dependencies

You need the Qdrant client, the Anthropic SDK, and a library for generating embeddings through Anthropic's voyage-3 model. Install them in one shot.

```bash
pip install qdrant-client anthropic voyageai
```

### Step 2: Set environment variables

Get your Anthropic API key from https://console.anthropic.com/settings/keys. Get your Voyage AI API key from https://dash.voyageai.com/api-keys (voyage-3 is the model we use for embeddings). If you run Qdrant locally via Docker, no API key is needed for Qdrant. For Qdrant Cloud, get your key from https://cloud.qdrant.io.

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
export VOYAGE_API_KEY="pa-..."
export QDRANT_URL="http://localhost:6333"
```

### Step 3: Start Qdrant locally

Pull and run the Qdrant Docker image. This gives you a vector store on port 6333 with no authentication required for local development.

```bash
docker run -d --name qdrant -p 6333:6333 -p 6334:6334 qdrant/qdrant:v1.12.1
```

### Step 4: Create a collection with payload indexes

Create a Qdrant collection sized for voyage-3 embeddings (1024 dimensions). Then create payload indexes on `user_id` and `document_date`. These indexes are what make filtered search fast. Without them, Qdrant scans every payload at query time.

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PayloadSchemaType
import os

client = QdrantClient(url=os.environ["QDRANT_URL"])

client.create_collection(
    collection_name="invoices",
    vectors_config=VectorParams(size=1024, distance=Distance.COSINE),
)

client.create_payload_index(
    collection_name="invoices",
    field_name="user_id",
    field_schema=PayloadSchemaType.KEYWORD,
)

client.create_payload_index(
    collection_name="invoices",
    field_name="document_date",
    field_schema=PayloadSchemaType.DATETIME,
)
```

### Step 5: Embed and upsert chunks with metadata

Embed a batch of invoice chunks using voyage-3. Each chunk carries a payload with `user_id`, `document_date`, and `doc_type`. In production you would batch millions. Here we insert a small set to prove the mechanism.

```python
import voyageai
from qdrant_client.models import PointStruct

vo = voyageai.Client(api_key=os.environ["VOYAGE_API_KEY"])

chunks = [
    {"text": "Invoice #4821. Total: $1,247.50. Services rendered in March 2026.", "user_id": "user_338", "document_date": "2026-03-15T00:00:00Z", "doc_type": "invoice"},
    {"text": "Invoice #4822. Total: $890.00. Services rendered in March 2026.", "user_id": "user_501", "document_date": "2026-03-18T00:00:00Z", "doc_type": "invoice"},
    {"text": "Refund notice for invoice #4801. Amount: $200.00. Issued March 2026.", "user_id": "user_338", "document_date": "2026-03-22T00:00:00Z", "doc_type": "refund"},
    {"text": "Invoice #4900. Total: $3,100.00. Services rendered in April 2026.", "user_id": "user_338", "document_date": "2026-04-10T00:00:00Z", "doc_type": "invoice"},
    {"text": "Billing policy update: net-30 terms effective March 2026.", "user_id": "global", "document_date": "2026-03-01T00:00:00Z", "doc_type": "policy"},
]

texts = [c["text"] for c in chunks]
embeddings = vo.embed(texts, model="voyage-3", input_type="document").embeddings

points = [
    PointStruct(
        id=i,
        vector=embeddings[i],
        payload={"text": chunks[i]["text"], "user_id": chunks[i]["user_id"], "document_date": chunks[i]["document_date"], "doc_type": chunks[i]["doc_type"]},
    )
    for i in range(len(chunks))
]

client.upsert(collection_name="invoices", points=points)
```

### Step 6: Query without filters (the broken version)

Embed the user's question. Search with no filter. Observe that multiple chunks come back, including ones belonging to the wrong user and wrong document types.

```python
query_text = "What was my invoice total for March 2026?"
query_embedding = vo.embed([query_text], model="voyage-3", input_type="query").embeddings[0]

unfiltered_results = client.query_points(
    collection_name="invoices",
    query=query_embedding,
    limit=5,
)

print("UNFILTERED RESULTS:")
for r in unfiltered_results.points:
    print(f"  score={r.score:.4f} user={r.payload['user_id']} text={r.payload['text'][:80]}")
```

### Step 7: Query with metadata filters (the correct version)

Now add a filter that restricts results to the requesting user and the target month. The filter uses a `must` clause combining a keyword match on `user_id` and a datetime range on `document_date`. We also filter `doc_type` to `invoice` to exclude refund notices.

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, DatetimeRange

filtered_results = client.query_points(
    collection_name="invoices",
    query=query_embedding,
    query_filter=Filter(
        must=[
            FieldCondition(key="user_id", match=MatchValue(value="user_338")),
            FieldCondition(key="doc_type", match=MatchValue(value="invoice")),
            FieldCondition(
                key="document_date",
                range=DatetimeRange(
                    gte="2026-03-01T00:00:00Z",
                    lt="2026-04-01T00:00:00Z",
                ),
            ),
        ]
    ),
    limit=1,
)

print("FILTERED RESULTS:")
for r in filtered_results.points:
    print(f"  score={r.score:.4f} user={r.payload['user_id']} text={r.payload['text']}")
```

### Step 8: Pass the filtered chunk to Claude for answer generation

Take the single correct chunk and send it to Claude with the original question. Claude generates the answer from verified context only.

```python
import anthropic

anthropic_client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

chunk_text = filtered_results.points[0].payload["text"]

response = anthropic_client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=256,
    messages=[
        {
            "role": "user",
            "content": f"Based on the following document chunk, answer the question.\n\nChunk: {chunk_text}\n\nQuestion: {query_text}",
        }
    ],
)

print("ANSWER:", response.content[0].text)
```

### Step 9: Measure latency

Wrap the filtered query in a timer. On a local Qdrant instance with five chunks this will be sub-millisecond. On Qdrant Cloud with ten million chunks and payload indexes, expect under 100ms. The payload index is what keeps it fast: Qdrant prunes the candidate set before computing cosine similarity.

```python
import time

start = time.perf_counter()
client.query_points(
    collection_name="invoices",
    query=query_embedding,
    query_filter=Filter(
        must=[
            FieldCondition(key="user_id", match=MatchValue(value="user_338")),
            FieldCondition(key="doc_type", match=MatchValue(value="invoice")),
            FieldCondition(
                key="document_date",
                range=DatetimeRange(
                    gte="2026-03-01T00:00:00Z",
                    lt="2026-04-01T00:00:00Z",
                ),
            ),
        ]
    ),
    limit=1,
)
elapsed_ms = (time.perf_counter() - start) * 1000
print(f"Filtered query latency: {elapsed_ms:.2f} ms")
```

## Breakage

Skip the metadata filters. The embedding for "invoice total March 2026" is semantically close to refund notices, billing policies, and invoices from other users. All of those chunks score above 0.85 cosine similarity. Claude receives a chunk belonging to `user_501` and reports $890.00 as the total. The answer is confident, well-formatted, and wrong. The user has no way to know. You have no way to know until someone complains. The failure is silent because embeddings have no concept of ownership or time.

```text
DIAGRAM: Failure mode without metadata filters
Caption: Without filters, Qdrant returns the highest-similarity chunk regardless of user or date, leading to a wrong answer.
Nodes:
1. User query - "What was my invoice total for March 2026?" (user_338)
2. Qdrant unfiltered search - returns top-1 by cosine similarity alone
3. Wrong chunk - Invoice #4822 belonging to user_501, score 0.93
4. Claude API - generates confident wrong answer from wrong chunk
Flow:
- User query vector goes to Qdrant with no filter
- Qdrant returns the wrong chunk (highest similarity, wrong user)
- Wrong chunk goes to Claude
- Claude returns "$890.00" to the user (incorrect)
```

## The fix

The fix is the filter from Step 7. It is not a post-processing step. It is not a reranker. The filter runs inside Qdrant's query engine before similarity scoring, which means it does not add latency. It subtracts candidates. Here it is extracted into a reusable function that takes the user context and returns the correct filter.

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, DatetimeRange

def build_retrieval_filter(user_id: str, doc_type: str, date_gte: str, date_lt: str) -> Filter:
    return Filter(
        must=[
            FieldCondition(key="user_id", match=MatchValue(value=user_id)),
            FieldCondition(key="doc_type", match=MatchValue(value=doc_type)),
            FieldCondition(
                key="document_date",
                range=DatetimeRange(gte=date_gte, lt=date_lt),
            ),
        ]
    )

retrieval_filter = build_retrieval_filter(
    user_id="user_338",
    doc_type="invoice",
    date_gte="2026-03-01T00:00:00Z",
    date_lt="2026-04-01T00:00:00Z",
)

fixed_results = client.query_points(
    collection_name="invoices",
    query=query_embedding,
    query_filter=retrieval_filter,
    limit=1,
)

chunk_text = fixed_results.points[0].payload["text"]
print(f"Correct chunk: {chunk_text}")
```

## Fixed state

```text
DIAGRAM: Retrieval pipeline with metadata filters in place
Caption: Filters prune candidates by user_id, doc_type, and date range before similarity scoring. Only valid chunks compete.
Nodes:
1. User query - "What was my invoice total for March 2026?" (user_338)
2. Orchestrator - builds filter from session context
3. Qdrant filtered search - prunes to user_338 + invoice + March 2026, then scores
4. Correct chunk - Invoice #4821, $1,247.50, user_338, March 2026
5. Claude API - generates answer from the correct chunk
Flow:
- User query enters Orchestrator
- Orchestrator constructs filter: user_338, invoice, 2026-03-01 to 2026-04-01
- Orchestrator sends vector + filter to Qdrant
- Qdrant returns one chunk: Invoice #4821
- Orchestrator sends chunk to Claude
- Claude returns "$1,247.50" to the user (correct)
```

## After

You have a RAG agent. A user asks: "What was my invoice total for March 2026?" Your agent embeds that query, constructs a metadata filter from the session context (user_id, date range, document type), and sends both to Qdrant. One chunk comes back. It is Invoice #4821, $1,247.50, belonging to user_338, dated March 2026. Claude reads that single chunk and returns the correct total. The query took 4ms on a local instance. On a ten-million-chunk Qdrant Cloud deployment with payload indexes, it takes under 80ms. The human alternative is writing a SQL query by hand, scanning results, verifying the right row, and reading the total. That takes minutes on a good day. The agent took milliseconds and got it right.

## Takeaway

Semantic similarity is not enough to retrieve the right chunk. It is only enough to retrieve a relevant chunk. The difference between relevant and right is metadata: who owns it, when it was created, what type it is. Encode that metadata at ingestion time, index it, and filter on it at query time. This pattern applies to every retrieval system where documents have owners, dates, or categories, which is every retrieval system that matters.
