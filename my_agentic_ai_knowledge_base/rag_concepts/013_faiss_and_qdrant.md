# FAISS and Qdrant Vector Stores

> From notebook: `009_beginner_vectorDB.ipynb`

## Why Not Just Loop Through Chunks?
Linear scan works for small sets but fails in production — cosine similarity over thousands of chunks is too slow. Vector DBs use k-NN / ANN algorithms to make this fast.

---

## FAISS (Facebook AI Similarity Search)

In-process, file-backed vector store. No server needed.

### Create and Persist
```python
from langchain_community.vectorstores.faiss import FAISS

# Build index from documents
vector_store = FAISS.from_documents(
    documents=chunks,
    embedding=embedding_model,
)

# Persist to disk
vector_store.save_local(folder_path=".vector_store/FAISS", index_name="data")

# Load back
vector_store = FAISS.load_local(
    folder_path=".vector_store/FAISS",
    index_name="data",
    embeddings=embedding_model,
    allow_dangerous_deserialization=True,
)
```

### Search
```python
results = vector_store.similarity_search_with_relevance_scores(
    query="financial results",
    k=20,
)
# returns list of (Document, score) tuples
```

### Filter by Metadata
```python
# Dict filter (exact match)
results = vector_store.similarity_search_with_relevance_scores(
    query=query,
    k=20,
    filter={"source": "my_doc.pdf"},
)

# Function filter (any condition)
results = vector_store.similarity_search_with_relevance_scores(
    query=query,
    k=20,
    filter=lambda meta: meta.get("page") > 5,
)
```

### Score Threshold (manual, FAISS)
FAISS doesn't support native threshold — filter results after retrieval:
```python
threshold = 0.73
relevant = [r for r in results if r[1] > threshold]
```

---

## Qdrant

Dedicated vector DB — in-memory, local file, or remote server.

### Create
```python
from langchain_community.vectorstores import Qdrant

# In-memory (testing)
vector_store = Qdrant.from_documents(
    chunks,
    embedding_model,
    location=":memory:",
    collection_name="my_documents",
)

# Persistent local
vector_store = Qdrant.from_documents(
    chunks,
    embedding_model,
    path="./qdrant_storage",
    collection_name="my_documents",
    force_recreate=True,
)
```

### Search with Native Score Threshold
Unlike FAISS, Qdrant supports threshold directly:
```python
results = vector_store.similarity_search_with_score(
    query=query,
    k=20,
    score_threshold=0.80,  # built-in filter
)
```

### Filter — Simple Dict
```python
results = vector_store.similarity_search_with_score(
    query=query,
    k=20,
    filter={"source": "my_doc.pdf"},
)
```

### Filter — Advanced Qdrant Query
```python
from qdrant_client import models as qdrant_models

custom_filter = qdrant_models.Filter(
    must=[
        qdrant_models.FieldCondition(
            key=f"{vector_store.metadata_payload_key}.source",
            match=qdrant_models.MatchValue(value="my_doc.pdf"),
        ),
        qdrant_models.FieldCondition(
            key=f"{vector_store.metadata_payload_key}.custom",
            range=qdrant_models.Range(gte=1000),
        ),
    ]
)

results = vector_store.similarity_search_with_score(
    query=query,
    k=20,
    filter=custom_filter,
)
```

### Fetch All Documents from Qdrant
```python
all_docs = vector_store.client.scroll(
    collection_name=vector_store.collection_name,
    offset=0,
    limit=vector_store.client.get_collection(vector_store.collection_name).points_count,
    with_payload=True,
    with_vectors=False,
)
```

---

## Adding Custom Metadata to Chunks
Metadata is stored alongside vectors and can be filtered at query time:
```python
for chunk in chunks:
    chunk.metadata["custom"] = len(chunk.page_content)
    chunk.metadata["department"] = "finance"
```

---

## Cosine vs Euclidean Distance
Both give the same rank order on normalized embeddings:

| Metric | Range | Higher = ? |
|---|---|---|
| Cosine similarity | 0–1 | More similar |
| Euclidean distance | 0–∞ | Less similar |

For normalized vectors (e.g. OpenAI ada-002), cosine and euclidean rankings are equivalent.

---

## FAISS vs Qdrant Comparison

| Feature | FAISS | Qdrant |
|---|---|---|
| Deployment | In-process | Separate server / in-memory |
| Persistence | Save/load files | File or remote |
| Score threshold | Manual post-filter | Native parameter |
| Advanced filters | Lambda function | Rich query DSL |
| Production ready | Moderate | Yes |
| Install | `faiss-cpu` | `langchain-qdrant` |
