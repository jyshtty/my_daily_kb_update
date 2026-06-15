# Reranking in RAG

## The Problem with Vector Retrieval Alone

Vector similarity is fast but imprecise — it finds chunks that are *roughly* related,
not necessarily the *most* relevant to the specific query.

```
Query: "What are LangGraph checkpointing backends?"

Retrieved (by similarity):
  1. "LangGraph is a library for building stateful agents"  ← related but not the answer
  2. "Checkpointing saves graph state for resumption"       ← closer
  3. "PostgresSaver and SqliteSaver are checkpointing backends" ← exact answer (ranked 3rd!)
```

Reranking re-scores and reorders these results using a more powerful model.

## How Reranking Works

```
Query + Candidate Chunks
    → Cross-Encoder model scores each (query, chunk) pair
    → Reorder by score
    → Pass top-k to LLM
```

Two-stage pipeline:
1. **Retrieve** fast with vector search (top 20-50 candidates)
2. **Rerank** accurately with cross-encoder (return top 3-5)

## Cross-Encoder vs Bi-Encoder

| | Bi-Encoder (Vector Search) | Cross-Encoder (Reranker) |
|---|---|---|
| How | Embed query and doc separately | Process query + doc together |
| Speed | Fast (pre-computed vectors) | Slow (computed at query time) |
| Accuracy | Lower | Higher |
| Use | First-stage retrieval | Second-stage reranking |

## Implementation with Cohere Reranker

```python
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever
from langchain_cohere import CohereRerank

reranker = CohereRerank(model="rerank-english-v3.0", top_n=3)

reranking_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20})
)

# Use exactly like a normal retriever
docs = reranking_retriever.invoke("What are checkpointing backends?")
```

## Implementation with Cross-Encoder (Local, Free)

```python
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

model = HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-base")
reranker = CrossEncoderReranker(model=model, top_n=3)

reranking_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20})
)
```

## When to Use Reranking

| Situation | Use Reranking? |
|---|---|
| Precision matters more than speed | Yes |
| Large document corpus with noisy retrieval | Yes |
| Low latency required | No |
| Small, clean document set | Probably not needed |

## Key Rule
Retrieve wide (k=20), rerank tight (top_n=3).
More candidates = better recall; reranker = better precision.

