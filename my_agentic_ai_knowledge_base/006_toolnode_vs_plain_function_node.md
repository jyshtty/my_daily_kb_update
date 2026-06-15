# ToolNode vs Plain Function Node in LangGraph

## ToolNode
```python
builder.add_node("tools", ToolNode(tools))
```
- Designed to handle **LLM tool call responses**
- When an LLM says "call this tool", ToolNode executes it automatically
- Expects `AIMessage` with `tool_calls` in state
- Part of the ReAct loop: LLM → tools → LLM → tools...
- The **LLM decides** which tool to call and when

## Plain Function Node
```python
workflow.add_node("booking", booking_agent)
```
- Just a **regular Python function** that runs unconditionally
- No LLM involvement — you define the logic
- Gets the state, does its work, returns updated state
- **You decide** when it runs via graph edges/conditions

## Comparison

| | ToolNode | Plain Function Node |
|---|---|---|
| **Who decides to run it?** | The LLM | You (via graph edges) |
| **Input** | AIMessage with tool_calls | State dict |
| **Use case** | Agentic loops | Deterministic workflows |
| **Pattern** | ReAct agent | Fixed workflow |

## Code Example

### ToolNode (Agentic)
```python
from langgraph.prebuilt import ToolNode

@tool
def search_docs(query: str) -> str:
    """Search documentation"""
    ...

tools = [search_docs]
builder.add_node("tools", ToolNode(tools))
builder.add_node("llm", call_llm)

# LLM decides when to call tools
builder.add_conditional_edges("llm", should_use_tool)
builder.add_edge("tools", "llm")  # loop back
```

### Plain Function Node (Workflow)
```python
def booking_agent(state):
    # Always runs when graph reaches this node
    return {"booking_status": "confirmed"}

workflow.add_node("booking", booking_agent)
workflow.add_edge("booking", "housekeeping")  # you define the path
```

## Real World Insight

In production, plain function nodes like `booking_agent` that call external APIs
should ideally be **tools registered with ToolNode** — so the LLM can dynamically
decide whether to book, cancel, or check availability based on user intent,
rather than always executing in a fixed order.

