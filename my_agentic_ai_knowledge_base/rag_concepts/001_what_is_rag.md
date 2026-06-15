# What is RAG (Retrieval Augmented Generation)

## The Problem RAG Solves

LLMs have two limitations:
- **Knowledge cutoff** — they do not know recent events
- **No private data** — they cannot access your documents, databases, or internal knowledge

RAG solves both by fetching relevant context at query time and injecting it into the prompt.

## How RAG Works

```
User Query
    → Retrieve relevant documents from a knowledge store
    → Inject documents into LLM prompt as context
    → LLM generates answer grounded in those documents
```

## Without RAG vs With RAG

| | Without RAG | With RAG |
|---|---|---|
| Knowledge source | LLM training data only | Your documents + LLM |
| Up-to-date? | No (cutoff date) | Yes (live retrieval) |
| Private data? | No | Yes |
| Hallucination risk | High | Lower |
| Explainability | Hard | Can cite sources |

## Simple RAG Example

```python
# 1. Retrieve
docs = vectorstore.similarity_search(user_query, k=3)

# 2. Augment
context = "
".join([d.page_content for d in docs])
prompt = f"Context: {context}

Question: {user_query}"

# 3. Generate
answer = llm.invoke(prompt)
```

## When to Use RAG
- Answering questions over private documents (PDFs, wikis, code)
- Customer support bots with product knowledge base
- Any use case where LLM needs domain-specific or recent knowledge

