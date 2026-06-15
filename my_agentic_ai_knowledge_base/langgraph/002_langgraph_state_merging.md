## LangGraph State Merging

**`search_web` returns a partial dict `{"search_results": results}` — LangGraph automatically merges this into the `ResearchState`. The `search_results` field (defined in `ResearchState` at line 29) gets updated, and `synthesize_report` then reads it from `state["search_results"]` on line 51.**

---

```python
from typing import TypedDict, List
from langgraph.graph import StateGraph, END


# Line 29 — ResearchState defines the shared state schema
class ResearchState(TypedDict):
    query: str
    search_results: List[str]   # <-- this field gets auto-merged
    report: str


# search_web returns a partial dict — LangGraph merges {"search_results": results} into ResearchState
def search_web(state: ResearchState) -> dict:
    query = state["query"]

    # Simulate web search results
    results = [
        f"Result 1 for '{query}': LangGraph is a library for building stateful agents.",
        f"Result 2 for '{query}': LangGraph uses a graph-based execution model.",
        f"Result 3 for '{query}': Nodes return partial dicts that are merged into state.",
    ]

    # Only return the fields you want to update — LangGraph merges this automatically
    return {"search_results": results}


# Line 51 — synthesize_report reads search_results from the merged state
def synthesize_report(state: ResearchState) -> dict:
    results = state["search_results"]   # reads the auto-merged field

    report = "## Research Report\n\n"
    for i, result in enumerate(results, 1):
        report += f"{i}. {result}\n"

    return {"report": report}


# Build the graph
builder = StateGraph(ResearchState)

builder.add_node("search_web", search_web)
builder.add_node("synthesize_report", synthesize_report)

builder.set_entry_point("search_web")
builder.add_edge("search_web", "synthesize_report")
builder.add_edge("synthesize_report", END)

graph = builder.compile()

# Run
initial_state: ResearchState = {
    "query": "LangGraph state management",
    "search_results": [],
    "report": ""
}

final_state = graph.invoke(initial_state)
print(final_state["report"])
```

---

### How State Merging Works

```
Initial State
─────────────────────────────────────────
{ query: "LangGraph state management",
  search_results: [],
  report: "" }

         │
         ▼
   [ search_web ]
   returns: {"search_results": ["Result 1...", "Result 2...", "Result 3..."]}
   LangGraph merges this → search_results field updated

         │
         ▼
Merged State
─────────────────────────────────────────
{ query: "LangGraph state management",
  search_results: ["Result 1...", "Result 2...", "Result 3..."],
  report: "" }

         │
         ▼
   [ synthesize_report ]
   reads state["search_results"] → builds report
   returns: {"report": "## Research Report\n\n1. Result 1..."}

         │
         ▼
Final State
─────────────────────────────────────────
{ query: "LangGraph state management",
  search_results: ["Result 1...", "Result 2...", "Result 3..."],
  report: "## Research Report\n\n..." }
```

---

### Key Takeaway

- Nodes **don't need to return the full state** — only the fields they update.
- LangGraph **automatically merges** the returned partial dict into the current state.
- Downstream nodes can immediately read any previously updated field from `state`.
