# Self-RAG (Self-Reflective RAG)

## What is Self-RAG?

Standard RAG always retrieves — even when the answer is already in the LLM training data.
Self-RAG adds a **reflection step**: the LLM critiques its own retrieval and generation.

Four decisions the LLM makes at each step:

| Token | Decision | Options |
|---|---|---|
| `Retrieve` | Should I retrieve at all? | yes / no / continue |
| `ISREL` | Is the retrieved chunk relevant? | relevant / irrelevant |
| `ISSUP` | Does the chunk support my answer? | fully / partially / no |
| `ISUSE` | Is my answer useful? | 1–5 score |

## The Flow

```
Query
  → Should I retrieve? (Retrieve token)
      NO  → answer directly from LLM knowledge
      YES → retrieve chunks
              → Is chunk relevant? (ISREL)
                  NO  → discard, try again
                  YES → generate answer with chunk
                          → Does chunk support answer? (ISSUP)
                          → Is answer useful? (ISUSE)
                          → if low score → re-retrieve or revise
```

## LangGraph Implementation Pattern

```python
from langgraph.graph import StateGraph

def should_retrieve(state):
    # LLM decides if retrieval is needed
    response = llm.invoke(f"Do you need external info to answer: {state[question]}? yes/no")
    return "retrieve" if "yes" in response.content else "generate"

def grade_documents(state):
    # LLM grades each retrieved doc for relevance
    filtered = []
    for doc in state["documents"]:
        grade = llm.invoke(f"Is this doc relevant to the question?
Doc: {doc.page_content}
Question: {state[question]}")
        if "relevant" in grade.content:
            filtered.append(doc)
    return {"documents": filtered}

def grade_generation(state):
    # LLM checks if answer is grounded in documents
    score = llm.invoke(f"Is this answer supported by the docs?
Answer: {state[answer]}
Docs: {state[documents]}")
    return "useful" if "yes" in score.content else "not_useful"

builder = StateGraph(State)
builder.add_node("retrieve", retrieve)
builder.add_node("grade_docs", grade_documents)
builder.add_node("generate", generate)
builder.add_conditional_edges("start", should_retrieve)
builder.add_conditional_edges("generate", grade_generation)
```

## Self-RAG vs Standard RAG

| | Standard RAG | Self-RAG |
|---|---|---|
| Always retrieves | Yes | No — only when needed |
| Checks relevance | No | Yes |
| Checks groundedness | No | Yes |
| Hallucination risk | Medium | Low |
| Latency | Low | High (multiple LLM calls) |
| Cost | Low | High |

## When to Use
- High-stakes answers where factual accuracy is critical (medical, legal, finance)
- When your corpus is noisy and retrieval often returns irrelevant chunks
- When you want the LLM to skip retrieval for simple factual questions it already knows

