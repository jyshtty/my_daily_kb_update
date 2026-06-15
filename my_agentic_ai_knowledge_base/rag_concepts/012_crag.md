# CRAG — Corrective RAG

## What is CRAG?

CRAG adds a **correction step** after retrieval — if retrieved docs are not relevant enough,
it falls back to web search before generating the answer.

Three retrieval confidence levels:
- **Correct** — docs are relevant → use them directly
- **Incorrect** — docs are irrelevant → discard, use web search instead
- **Ambiguous** — mixed relevance → combine docs + web search

## Flow

```
Query → Retrieve from vector store
    → Grade relevance (LLM or cross-encoder)
        CORRECT   → generate answer from docs
        INCORRECT → web search → generate from web results
        AMBIGUOUS → refine query → web search → combine with docs → generate
```

## Key Difference from Self-RAG

| | Self-RAG | CRAG |
|---|---|---|
| Focus | Should I retrieve? Is answer grounded? | Are retrieved docs good enough? |
| Fallback | Re-retrieve or answer without docs | Web search |
| Complexity | Higher (4 reflection tokens) | Medium |

## LangGraph Implementation Pattern

```python
from tavily import TavilyClient

tavily = TavilyClient()

def grade_documents(state):
    relevant = []
    for doc in state["documents"]:
        score = llm.invoke(
            f"Is this relevant to the question?
Doc:{doc.page_content}
Question:{state[question]}"
        )
        if "yes" in score.content.lower():
            relevant.append(doc)
    
    if len(relevant) == 0:
        return {**state, "web_search_needed": True, "documents": []}
    elif len(relevant) < len(state["documents"]) / 2:
        return {**state, "web_search_needed": True, "documents": relevant}
    else:
        return {**state, "web_search_needed": False, "documents": relevant}

def web_search(state):
    results = tavily.search(query=state["question"])
    web_docs = [Document(page_content=r["content"]) for r in results["results"]]
    return {"documents": state["documents"] + web_docs}

def route_after_grading(state):
    return "web_search" if state["web_search_needed"] else "generate"

builder.add_conditional_edges("grade_docs", route_after_grading)
builder.add_edge("web_search", "generate")
```

## When to Use

| Situation | Use CRAG? |
|---|---|
| Your vector store may have gaps in coverage | Yes |
| Questions may require up-to-date information | Yes |
| Fully offline / no internet access allowed | No |
| Corpus is comprehensive and always relevant | Probably not needed |

## Key Rule
CRAG is a safety net — when your knowledge base is incomplete or stale,
fall back to the web rather than hallucinate.

