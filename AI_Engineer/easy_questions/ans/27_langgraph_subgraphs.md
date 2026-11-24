# Q27. What are subgraphs, and how do state and checkpoints work with them?

[← Back to questions](../questions.md)

## TL;DR

- A **subgraph** is a **compiled graph used as a node** (or called from a node) inside another graph. Use them for **encapsulation, reuse, multi-agent systems** (each agent is a subgraph), and team ownership.
- **Two ways to connect them:**
  1. **Add the compiled subgraph directly as a node** (`builder.add_node("sub", subgraph)`) when parent and child **share state keys**. Shared keys flow in and out automatically.
  2. **Call the subgraph inside a node function** when the schemas **differ**. You map the parent state to the subgraph input and its output back to the parent.
- **Checkpoints:** compile **only the parent** with a checkpointer and it **propagates** to subgraphs. Subgraph checkpoints get their own **`checkpoint_ns`** (namespace). Use `compile(checkpointer=True)` for a subgraph that keeps **its own memory across calls**. `checkpointer=False` disables it.
- `get_state(config, subgraphs=True)`, `stream(..., subgraphs=True)` and interrupts all work through subgraphs. An interrupt inside a subgraph rises to the parent, and resuming **re-runs the parent node that called the subgraph**.
- `Command(goto=..., graph=Command.PARENT)` lets a subgraph node jump to a node in the **parent** graph. This is how agent handoffs work.

## Detailed Answer

### 1. Why subgraphs

```
Parent graph (orchestrator)
┌──────────────────────────────────────────────────────────────┐
│ START ─► router ─► [research_agent] ─► [writer_agent] ─► END │
│                      │ subgraph          │ subgraph          │
│                      ▼                   ▼                   │
│              ┌───────────────┐   ┌───────────────┐           │
│              │ plan→search→  │   │ draft→review  │           │
│              │ read→summarize│   │   (loop)      │           │
│              └───────────────┘   └───────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

- **Encapsulation**: private state (scratch messages) stays inside the subgraph.
- **Reuse**: the same "research" subgraph can be used in several products.
- **Independent development and testing**: each subgraph can be invoked on its own.
- **Multi-agent**: supervisors and swarms are built from agent subgraphs.

### 2. Pattern A: shared keys (add as a node)

```python
import operator
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END

class SubState(TypedDict):
    topic: str                                     # shared with parent
    notes: Annotated[list[str], operator.add]      # shared with parent
    scratch: list[str]                             # private to subgraph

def search(s: SubState):
    return {"scratch": [f"raw results for {s['topic']}"]}

def summarize(s: SubState):
    return {"notes": [f"summary of {len(s['scratch'])} results"]}

sub = StateGraph(SubState)
sub.add_node(search); sub.add_node(summarize)
sub.add_edge(START, "search"); sub.add_edge("search", "summarize"); sub.add_edge("summarize", END)
research_subgraph = sub.compile()

class ParentState(TypedDict):
    topic: str
    notes: Annotated[list[str], operator.add]
    report: str

def write(s: ParentState):
    return {"report": " | ".join(s["notes"])}

p = StateGraph(ParentState)
p.add_node("research", research_subgraph)        # compiled graph as node
p.add_node("write", write)
p.add_edge(START, "research"); p.add_edge("research", "write"); p.add_edge("write", END)
app = p.compile()
print(app.invoke({"topic": "vector DBs", "notes": []}))
```

The parent's `topic` and `notes` go into the subgraph. When it finishes, the subgraph's values for keys **that exist in the parent schema** are written back through the parent's reducers. `scratch` never appears in the parent.

### 3. Pattern B: different schemas (invoke from a node)

```python
class AgentState(TypedDict):          # subgraph schema - no overlap with parent
    query: str
    messages: list
    answer: str

agent_graph = build_agent(AgentState).compile()

class ParentState(TypedDict):
    user_question: str
    final_answer: str

def call_agent(state: ParentState):
    out = agent_graph.invoke({"query": state["user_question"], "messages": []})  # map in
    return {"final_answer": out["answer"]}                                     # map out

p = StateGraph(ParentState)
p.add_node("agent", call_agent)
p.add_edge(START, "agent"); p.add_edge("agent", END)
```

Even though the subgraph is called inside a function, LangGraph still treats it as a subgraph for checkpointing, streaming, and interrupts, because the call inherits the runnable config.

| | Add as node | Call from node |
|---|---|---|
| Schema requirement | Overlapping keys | Any |
| Mapping | Automatic (shared keys) | Manual (you transform) |
| Multiple subgraph calls per node | No | Yes (e.g. loop over items) |
| Typical use | Shared `messages` multi-agent | Encapsulated agents with private schemas |

### 4. Checkpointing with subgraphs

```python
from langgraph.checkpoint.memory import InMemorySaver

app = p.compile(checkpointer=InMemorySaver())   # only the parent gets a checkpointer
```

- The checkpointer **propagates** to child graphs. Subgraph checkpoints are stored under a `checkpoint_ns` like `"research:<task_id>"`.
- **Default** (`checkpointer=None` on the subgraph) gives **per-invocation** persistence. Within one parent run the subgraph is durable (it can interrupt and resume), but each new call starts with a fresh subgraph state.
- `sub.compile(checkpointer=True)` makes the subgraph keep **its own state across invocations** within the parent thread. This is useful when an agent subgraph should remember its own conversation history over turns.
- `sub.compile(checkpointer=False)` means no checkpointing for the subgraph, even if the parent has one.
- Don't give a subgraph a **separate** saver instance unless you really want an isolated store.

#### Inspecting subgraph state

```python
cfg = {"configurable": {"thread_id": "1"}}
app.invoke({"topic": "x", "notes": []}, cfg)

parent = app.get_state(cfg, subgraphs=True)
for task in parent.tasks:
    if task.state:                     # StateSnapshot of the subgraph (only while paused/in-flight)
        print(task.name, task.state.values)
```

Subgraph state is only visible through `get_state(..., subgraphs=True)` while the subgraph is **interrupted or in progress**. After completion, its results live in the parent state.

#### Updating subgraph state when paused

```python
sub_cfg = parent.tasks[0].state.config      # includes checkpoint_ns
app.update_state(sub_cfg, {"scratch": ["human-edited results"]})
app.invoke(None, cfg)
```

### 5. Interrupts in subgraphs

```python
from langgraph.types import interrupt, Command

def summarize(s: SubState):
    ok = interrupt({"review": s["scratch"]})        # inside subgraph
    return {"notes": [f"approved={ok}"]}
```

- The interrupt surfaces at the **parent** level (`__interrupt__` in the parent output).
- Resume through the parent: `app.invoke(Command(resume=True), cfg)`.
- **Re-execution caveat:** on resume, the **parent node that invoked the subgraph runs again from its beginning**, and the subgraph node that interrupted re-runs too. Code in the parent node before the `subgraph.invoke(...)` call must be idempotent.

### 6. Navigating to the parent: handoffs

```python
def agent_node(state) -> Command:
    return Command(
        goto="billing_agent",          # a node in the PARENT graph
        update={"messages": [...]},    # keys must exist in parent schema (with reducer if shared)
        graph=Command.PARENT,
    )
```

`langgraph-swarm` and `langgraph-supervisor` handoff tools are built on this.

### 7. Streaming subgraphs

```python
for ns, chunk in app.stream({"topic": "x", "notes": []}, stream_mode="updates", subgraphs=True):
    print(ns, chunk)
# () {'research': {...}}                              <- parent
# ('research:3f2a...',) {'search': {'scratch': [...]}} <- inside subgraph
```

## Common Pitfalls

- **No overlapping keys when adding as a node.** Nothing flows in or out. Use Pattern B.
- **A shared key without a reducer in the parent**, while the subgraph also returns it after parallel work, causes conflicts.
- **Compiling subgraphs with their own `InMemorySaver()`.** Usually unnecessary and confusing. Let the parent's checkpointer propagate.
- **Expecting subgraph memory across turns by default.** It's per-invocation unless the subgraph is compiled with `checkpointer=True`.
- **Side effects in the parent node before calling a subgraph that may interrupt.** They get duplicated on resume.
- **Huge shared `messages` between parent and agent subgraphs.** Context bloat. Return only summaries.

## Likely Follow-up Questions

1. **When would you use a subgraph rather than more nodes?** For reuse, private state, an independent agent loop, a separate team or owner, or for testing it in isolation.
2. **How do you see what happened inside a subgraph?** `stream(..., subgraphs=True)`, `get_state(cfg, subgraphs=True)` while paused, and LangSmith nested traces.
3. **Can subgraphs run in parallel?** Yes. Use multiple edges or `Send` to the subgraph node, and each gets its own namespace.
4. **How do multi-agent handoffs use subgraphs?** Each agent is a subgraph. Handoff tools return `Command(goto=other_agent, graph=Command.PARENT)`.
5. **What is `checkpoint_ns`?** The namespace that separates subgraph checkpoints from the parent's within the same thread.

[← Back to questions](../questions.md)
