# Parent Document Retriever

## The Problem

Chunking creates a dilemma:
- **Small chunks** = precise retrieval but lose surrounding context
- **Large chunks** = rich context but noisy retrieval

```
Small chunk retrieved: "The function returns None."
← accurate match, but reader has no idea what function or why
```

## The Solution: Retrieve Small, Return Large

Store two levels of chunks:
1. **Child chunks** (small) — used for retrieval/matching
2. **Parent chunks** (large) — returned to the LLM for context

```
Index:   [small chunk 1] [small chunk 2] [small chunk 3]
                  ↓               ↓               ↓
Storage: [        large parent document            ]

Query → match small chunk → return its full parent
```

## Implementation

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Small chunks for retrieval
child_splitter = RecursiveCharacterTextSplitter(chunk_size=200)

# Large chunks returned to LLM
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

store = InMemoryStore()  # stores parent docs (use RedisStore in prod)

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

retriever.add_documents(docs)
results = retriever.invoke("your query")  # returns parent chunks
```

## Variant: Full Document Retrieval

Return the entire original document instead of a parent chunk:

```python
retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    # no parent_splitter = returns full original document
)
```

## When to Use

| Situation | Use Parent Doc Retriever? |
|---|---|
| Dense technical docs where context around a fact matters | Yes |
| Legal/medical documents where sentences reference each other | Yes |
| Short standalone FAQs | No — chunks are already self-contained |
| Very large documents (books) | Careful — parent may exceed context window |

## Key Rule
Match on small. Answer from large.

