# Vector Stores in RAG

## What is a Vector Store?

A database optimized for storing and searching **embedding vectors** using similarity metrics (cosine, dot product, euclidean).

## Common Vector Stores

### FAISS (Local, In-Memory)
```python
from langchain_community.vectorstores import FAISS
db = FAISS.from_documents(chunks, embeddings)
db.save_local("faiss_index")          # persist to disk
db = FAISS.load_local("faiss_index", embeddings)  # reload
```
- Free, fast, no setup
- Not scalable beyond single machine
- Good for: prototyping, local dev

### Chroma (Local, Persistent)
```python
from langchain_community.vectorstores import Chroma
db = Chroma.from_documents(chunks, embeddings, persist_directory="./chroma_db")
```
- Persistent by default
- Easy to run locally or as a server
- Good for: small-to-medium production apps

### Pinecone (Cloud, Managed)
```python
from langchain_pinecone import PineconeVectorStore
db = PineconeVectorStore.from_documents(chunks, embeddings, index_name="my-index")
```
- Fully managed, scales automatically
- Pay-per-use
- Good for: large-scale production

### pgvector (Postgres Extension)
```python
from langchain_postgres import PGVector
db = PGVector.from_documents(chunks, embeddings, connection=conn_str)
```
- Adds vector search to existing Postgres
- Great if you already use Postgres
- Good for: teams who want one database

## Comparison

| Store | Setup | Scale | Cost | Best For |
|---|---|---|---|---|
| FAISS | None | Single machine | Free | Prototyping |
| Chroma | Minimal | Small-medium | Free | Local dev/prod |
| Pinecone | Cloud signup | Massive | Paid | Large-scale prod |
| pgvector | Postgres | Large | Infrastructure | Existing Postgres users |

## Similarity Metrics

| Metric | When to use |
|---|---|
| **Cosine** | Text embeddings (default choice) |
| **Dot product** | Normalized embeddings, faster |
| **Euclidean** | When magnitude matters |

