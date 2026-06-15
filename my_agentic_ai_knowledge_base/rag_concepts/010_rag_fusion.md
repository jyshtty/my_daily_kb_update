# RAG Fusion (Multi-Query + Reciprocal Rank Fusion)

## The Problem

A single query retrieves from one angle only.
Different phrasings of the same question retrieve different — sometimes better — chunks.

## What RAG Fusion Does

1. Generate multiple query variants using LLM
2. Retrieve separately for each variant
3. Merge results using **Reciprocal Rank Fusion (RRF)** — a scoring algorithm that rewards chunks appearing in multiple result lists

```
Original query: "how to persist LangGraph state"
    ↓ LLM generates variants
Query 1: "LangGraph checkpointing options"
Query 2: "saving agent state between sessions in LangGraph"
Query 3: "LangGraph memory persistence backends"
    ↓ retrieve for each
    ↓ RRF merges and reranks
Final: top chunks that appeared across multiple retrievals
```

## Reciprocal Rank Fusion Formula

```
RRF_score(chunk) = Σ 1 / (k + rank_in_list_i)
```
- k = 60 (constant, reduces impact of very high ranks)
- A chunk ranked #1 in two lists scores higher than #1 in just one

## Implementation

```python
from langchain.retrievers import MergerRetriever
from langchain_openai import ChatOpenAI

def generate_queries(question: str, n: int = 4) -> list[str]:
    prompt = f"""Generate {n} different search queries for: {question}
    Output one query per line."""
    response = llm.invoke(prompt)
    return response.content.strip().split("
")

def reciprocal_rank_fusion(results: list[list], k: int = 60):
    scores = {}
    for result_list in results:
        for rank, doc in enumerate(result_list):
            key = doc.page_content
            scores[key] = scores.get(key, 0) + 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)

def rag_fusion_retrieve(question: str):
    queries = generate_queries(question)
    all_results = [retriever.invoke(q) for q in queries]
    return reciprocal_rank_fusion(all_results)
```

## RAG Fusion vs Multi-Query Retriever

| | Multi-Query | RAG Fusion |
|---|---|---|
| Query generation | Yes | Yes |
| Result merging | Simple dedup | RRF scoring |
| Ranking | Original vector score | Cross-list frequency |
| Quality | Good | Better |

## When to Use

- Ambiguous or broad queries with multiple valid interpretations
- When single-query retrieval consistently misses relevant chunks
- Research-style questions where completeness matters

## Tradeoff
Multiple LLM calls for query generation + multiple retrieval calls = higher latency and cost.
Best suited for quality-critical applications, not real-time chat.

