# Retrieval Strategies in RAG

## 1. Similarity Search (Default)
Returns top-k chunks most similar to the query vector.
```python
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})
```
- Fast, simple
- May return redundant/similar chunks

## 2. MMR — Maximal Marginal Relevance
Balances relevance AND diversity — avoids returning near-duplicate chunks.
```python
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 4, "fetch_k": 20}
)
```
- fetch_k: candidates to consider
- k: final results returned
- Good for: broad questions where diversity matters

## 3. Hybrid Search (Keyword + Semantic)
Combines BM25 (keyword) + vector (semantic) search.
```python
from langchain.retrievers import BM25Retriever, EnsembleRetriever

bm25 = BM25Retriever.from_documents(docs)
semantic = vectorstore.as_retriever()

retriever = EnsembleRetriever(
    retrievers=[bm25, semantic],
    weights=[0.4, 0.6]
)
```
- Best of both worlds: exact keyword match + semantic understanding
- Good for: technical docs with specific terms (function names, error codes)

## 4. Multi-Query Retrieval
LLM generates multiple rephrased versions of the query → retrieves for each → deduplicates.
```python
from langchain.retrievers.multi_query import MultiQueryRetriever
retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(), llm=llm
)
```
- Handles ambiguous or poorly worded queries
- Costs more (multiple LLM calls)

## 5. Contextual Compression
Compresses retrieved chunks — keeps only the sentences relevant to the query.
```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

compressor = LLMChainExtractor.from_llm(llm)
retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever()
)
```

## Quick Reference

| Strategy | Best For | Cost |
|---|---|---|
| Similarity | General use | Low |
| MMR | Diverse results needed | Low |
| Hybrid | Technical/keyword-heavy docs | Medium |
| Multi-Query | Ambiguous queries | High (extra LLM calls) |
| Compression | Noisy/long chunks | High (extra LLM calls) |

