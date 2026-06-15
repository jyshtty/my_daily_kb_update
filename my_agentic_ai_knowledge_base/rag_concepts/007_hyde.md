# HyDE — Hypothetical Document Embeddings

## The Problem with Standard RAG

Standard RAG embeds the **user query** and searches for similar chunks.
But queries are short and vague — they often do not match the style of documents.

```
Query:   "how to handle errors in async python"   ← short, informal
Document: "Exception handling in asynchronous coroutines requires..."  ← formal, detailed
```
The embeddings may not align well → wrong chunks retrieved.

## What HyDE Does

Instead of embedding the query, ask the LLM to **generate a hypothetical answer**,
then embed THAT. The hypothetical answer looks more like a real document → better match.

```
Query → LLM generates hypothetical answer → embed hypothetical answer → retrieve real docs
```

## Why It Works

The hypothetical answer uses the same vocabulary and style as real documents,
so its embedding lands closer to relevant chunks in vector space.

## Implementation

```python
from langchain.chains import HypotheticalDocumentEmbedder
from langchain_openai import OpenAIEmbeddings, ChatOpenAI

base_embeddings = OpenAIEmbeddings()
llm = ChatOpenAI()

# Wrap embeddings with HyDE
hyde_embeddings = HypotheticalDocumentEmbedder.from_llm(
    llm=llm,
    base_embeddings=base_embeddings,
    prompt_key="web_search"  # or "sci_fact", "fiqa", custom
)

# Use exactly like normal embeddings
vectorstore = FAISS.from_documents(chunks, hyde_embeddings)
retriever = vectorstore.as_retriever()
```

## Manual HyDE (More Control)

```python
hyde_prompt = ChatPromptTemplate.from_template(
    "Write a detailed answer to this question as if it were a documentation excerpt:
{question}"
)

def hyde_retrieve(question):
    hypothetical_doc = llm.invoke(hyde_prompt.format(question=question))
    return retriever.invoke(hypothetical_doc.content)  # embed the answer, not the query
```

## When to Use HyDE

| Situation | Use HyDE? |
|---|---|
| Short, vague queries | Yes |
| Technical docs with formal language | Yes |
| Queries already long and descriptive | No — adds latency for no gain |
| Latency-sensitive applications | No — extra LLM call |

## Tradeoff
- **Better retrieval quality** for vague queries
- **Extra LLM call** = more latency and cost

