# Multi-Agent Patterns

## Why Multi-Agent?

A single agent with too many tools becomes unfocused and unreliable.
Split responsibilities across specialized agents for better performance.

## Pattern 1: Supervisor

One LLM acts as a router — it reads the user request and delegates to the right agent.

```
User → Supervisor LLM
           ↓
    which agent handles this?
    ├── research_agent  (web search, summarization)
    ├── coding_agent    (write/debug code)
    └── data_agent      (SQL queries, charts)
```

```python
def supervisor(state):
    # LLM decides which agent to call next
    response = llm.invoke(state["messages"])
    return {"next": response.next_agent}

builder.add_conditional_edges("supervisor", lambda s: s["next"])
```

## Pattern 2: Subgraph

Each agent is a fully compiled graph nested inside a parent graph.

```python
research_graph = research_builder.compile()
coding_graph = coding_builder.compile()

# Add as nodes in parent graph
parent_builder.add_node("researcher", research_graph)
parent_builder.add_node("coder", coding_graph)
```

## Pattern 3: Parallel (Map-Reduce)

Run multiple agents simultaneously, then aggregate results.

```
                 ┌── agent_1 ──┐
User Query ──────├── agent_2 ──┼──→ aggregator → Final Answer
                 └── agent_3 ──┘
```

## Comparison

| Pattern | Best For | Complexity |
|---|---|---|
| **Supervisor** | Dynamic routing based on intent | Medium |
| **Subgraph** | Encapsulated specialist agents | Medium |
| **Parallel** | Independent tasks that can run simultaneously | High |

## Key Rule
- Each agent should have a **single, clear responsibility**
- Agents communicate via **shared state**, not direct calls
- Use supervisor when routing logic is complex or dynamic

