# Workflow vs Agent in LangGraph

## Workflow (Deterministic)
- Path is **hardcoded** — you define every edge
- Nodes always execute in a fixed order
- No LLM involved in routing decisions
- Predictable, fast, easy to debug

```python
workflow.add_edge("booking", "housekeeping")
workflow.add_edge("housekeeping", "customer_service")
# Always: booking → housekeeping → customer_service
```

## Agent (Dynamic)
- Path is **LLM-driven** — the LLM decides what to do next
- LLM picks which tools to call based on user intent
- Loops until LLM decides it has enough information
- Flexible, intelligent, but less predictable

```python
agent = create_react_agent(llm, tools=[book_room, cancel_room, check_availability])
# LLM decides: should I book? cancel? check first?
```

## Comparison

| | Workflow | Agent |
|---|---|---|
| **Who controls flow?** | You (edges) | LLM |
| **Path** | Fixed | Dynamic |
| **Tools** | Plain function nodes | ToolNode |
| **Predictability** | High | Low |
| **Flexibility** | Low | High |
| **Use case** | Known, repeatable processes | Open-ended user queries |
| **Example** | Hotel booking pipeline | Customer support bot |

## When to Use Each

**Use Workflow when:**
- Steps are always the same regardless of input
- You need full control and auditability
- Process is well-defined (e.g., ETL pipeline, order fulfillment)

**Use Agent when:**
- User input varies and you cannot predict the path upfront
- LLM needs to reason about what to do next
- Multiple tools exist and only some are needed per query

## Hybrid Approach (Common in Production)

```
User Request
    → Agent (LLM decides high-level intent)
        → calls booking_workflow tool (deterministic sub-process)
        → calls housekeeping_workflow tool (deterministic sub-process)
    → Agent composes final response
```

Agents orchestrate; workflows execute. Use both together for robust systems.

