# LangGraph Checkpointing and Human-in-the-Loop

## What is Checkpointing?

Checkpointing saves the graph state after every node execution so it can be:
- **Resumed** after interruption
- **Inspected** at any point
- **Replayed** from any step

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# Each run needs a thread_id to track state
config = {"configurable": {"thread_id": "user-123"}}
graph.invoke({"messages": [...]}, config=config)
```

## Human-in-the-Loop

Pause the graph and wait for human approval before continuing.

```python
# Interrupt BEFORE a node runs
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["book_flight"]  # pause here
)

# Run until interrupt
graph.invoke(input, config)

# Human reviews state, then resumes
graph.invoke(None, config)  # None = resume from checkpoint
```

## Why It Matters

| Without Checkpointing | With Checkpointing |
|---|---|
| Agent crashes = start over | Resume from last saved step |
| No human oversight | Pause for approval |
| Single session only | Multi-session conversations |
| No audit trail | Full state history |

## Persistence Options

```python
from langgraph.checkpoint.memory import MemorySaver      # in-memory (dev)
from langgraph.checkpoint.sqlite import SqliteSaver      # SQLite (local prod)
from langgraph.checkpoint.postgres import PostgresSaver  # Postgres (prod)
```

## Key Use Cases
- **Approval workflows**: pause before sending email, making payment
- **Long-running tasks**: resume multi-day processes
- **Debugging**: inspect state at any node
- **Multi-turn chat**: remember conversation across sessions

