# Q18. How would you maintain shared state and track the work completed by each agent?

[← Back to questions](../questions.md)

## TL;DR

- Use a single **typed graph state** (TypedDict/Pydantic) as the **shared blackboard**. Put the global facts every agent needs there (goal, plan, task statuses, final outputs).
- Give each agent its **own output key** (`research_result`, `analysis_result`) or a **namespaced dict** merged by a custom reducer, so agents never overwrite each other.
- Keep **private working state** (scratch messages, tool chatter) inside **subgraphs** with their own schema, and return only a summary to the parent.
- Track progress with a **task ledger**: a list of `{task_id, agent, status, output, started_at, finished_at}` entries, appended via a reducer and updated by ID.
- Use **checkpoint history** (`get_state_history`, `metadata["writes"]` shows which node wrote what) plus **LangSmith/OTel tracing** (per-agent spans, tokens, latency) for audit and debugging. Use the **Store** for state that has to outlive a thread.

## Detailed Answer

### 1. Layers of state

```
┌──────────────────────── Long-term Store (cross-thread) ─────────────────────────┐
│  namespace ("user", uid) -> preferences, past incident summaries                  │
└───────────────────────────────────────────────────────────────────────────────────┘
┌──────────────────────── Shared graph state (one thread) ──────────────────────────┐
│ goal | plan | tasks (ledger) | agent_outputs{agent: result} | messages | errors    │
└───────────────────────────────────────────────────────────────────────────────────┘
      ▲ summary only                    ▲ summary only
┌─────┴──────────────┐          ┌───────┴────────────┐
│ Researcher subgraph│          │ Coder subgraph      │
│ private: messages, │          │ private: messages,  │
│ search_results     │          │ code, test_output   │
└────────────────────┘          └─────────────────────┘
```

| Layer | Lifetime | Mechanism | Holds |
|---|---|---|---|
| Private agent state | One agent invocation | Subgraph state schema | Tool calls, scratch reasoning |
| Shared state | One thread/run | Parent graph state + reducers | Plan, ledger, outputs |
| Checkpoint history | Thread lifetime | Checkpointer | Every super-step snapshot |
| Long-term memory | Across threads | `BaseStore` | User facts, learned procedures |
| Traces | Ops retention | LangSmith / OTel | Inputs, outputs, tokens, latency per agent |

### 2. Designing the shared state

```python
import operator
from datetime import datetime, timezone
from typing import Annotated, Literal, TypedDict
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages

Status = Literal["pending", "in_progress", "done", "failed"]

class Task(TypedDict):
    id: str
    agent: str
    description: str
    status: Status
    output: str | None
    error: str | None
    updated_at: str

def upsert_tasks(left: list[Task] | None, right: list[Task] | None) -> list[Task]:
    """Ledger reducer: append new tasks, update existing ones by id."""
    by_id = {t["id"]: t for t in (left or [])}
    for t in right or []:
        by_id[t["id"]] = {**by_id.get(t["id"], {}), **t}
    return list(by_id.values())

def merge_outputs(left: dict | None, right: dict | None) -> dict:
    return {**(left or {}), **(right or {})}

class TeamState(TypedDict):
    goal: str
    messages: Annotated[list[AnyMessage], add_messages]      # user-facing conversation
    tasks: Annotated[list[Task], upsert_tasks]                # task ledger
    agent_outputs: Annotated[dict[str, dict], merge_outputs]  # per-agent results
    events: Annotated[list[dict], operator.add]               # append-only audit log
    errors: Annotated[list[str], operator.add]
```

Why this works:

- **Parallel safety**: every shared key that more than one agent writes has a reducer, so concurrent writes never raise `InvalidUpdateError`.
- **Attribution**: `agent_outputs["researcher"]` and the `agent` field on each task make clear who did what.
- **Idempotent updates**: the ledger upserts by `id`, so retries don't duplicate entries.

### 3. A worker that records its own work

```python
def now() -> str:
    return datetime.now(timezone.utc).isoformat()

def make_worker(name: str, agent):
    def node(state: TeamState):
        task = next(t for t in state["tasks"] if t["agent"] == name and t["status"] == "pending")
        try:
            result = agent.invoke({"messages": [("user", task["description"])]})
            answer = result["messages"][-1].content
            return {
                "tasks": [{**task, "status": "done", "output": answer, "updated_at": now()}],
                "agent_outputs": {name: {"task_id": task["id"], "summary": answer}},
                "events": [{"agent": name, "event": "task_done", "task": task["id"], "at": now()}],
            }
        except Exception as e:
            return {
                "tasks": [{**task, "status": "failed", "error": str(e), "updated_at": now()}],
                "errors": [f"{name}: {e}"],
                "events": [{"agent": name, "event": "task_failed", "task": task["id"], "at": now()}],
            }
    return node
```

The supervisor or planner reads `state["tasks"]` to decide what's next:

```python
def next_step(state: TeamState):
    pending = [t for t in state["tasks"] if t["status"] == "pending"]
    if not pending:
        return "synthesize"
    return pending[0]["agent"]
```

### 4. Private state via subgraphs

```python
from langgraph.graph import StateGraph, START, END

class ResearchPrivate(TypedDict):
    query: str
    messages: Annotated[list[AnyMessage], add_messages]   # private scratch
    sources: list[str]
    summary: str

research_sub = build_research_graph(ResearchPrivate).compile()

def research_node(state: TeamState):
    task = next(t for t in state["tasks"] if t["agent"] == "researcher" and t["status"] == "pending")
    out = research_sub.invoke({"query": task["description"], "messages": []})
    # Only the distilled result crosses into shared state
    return {
        "agent_outputs": {"researcher": {"summary": out["summary"], "sources": out["sources"]}},
        "tasks": [{**task, "status": "done", "output": out["summary"], "updated_at": now()}],
    }
```

If you add the compiled subgraph directly as a node (`builder.add_node("research", research_sub)`), keys that **share names** with the parent are passed in and out automatically. The rest stay private.

### 5. Tracking with checkpoints

Every super-step is checkpointed, and `metadata["writes"]` records what each node returned:

```python
cfg = {"configurable": {"thread_id": "project-7"}}
for snap in reversed(list(app.get_state_history(cfg))):
    writes = snap.metadata.get("writes") or {}
    for node, update in writes.items():
        print(snap.metadata["step"], node, list((update or {}).keys()))
# 1 planner ['tasks']
# 2 researcher ['tasks', 'agent_outputs', 'events']
# 2 coder ['tasks', 'agent_outputs', 'events']      <- parallel, same step
# 3 synthesize ['messages']
```

Also useful:

- `app.get_state(cfg, subgraphs=True)` inspects nested agent state.
- `app.stream(..., stream_mode="updates", subgraphs=True)` gives a live feed of `(namespace, {node: update})`, which is ideal for a progress UI.
- `stream_mode="custom"` with `get_stream_writer()` lets an agent emit progress events like `{"agent": "coder", "progress": "tests 3/10"}`.

### 6. Tracing

```python
app.invoke(inputs, {
    "configurable": {"thread_id": "project-7"},
    "run_name": "team-run",
    "tags": ["multi-agent", "prod"],
    "metadata": {"user_id": "u-42"},
})
```

With `LANGSMITH_TRACING=true`, each node and sub-agent appears as a nested run with inputs, outputs, tool calls, token usage, and latency. Filter by node name to see per-agent cost and error rates.

### 7. Cross-thread memory with the Store

```python
from langgraph.store.base import BaseStore

def researcher(state: TeamState, *, store: BaseStore):
    prior = store.search(("org", "acme", "research"), query=state["goal"], limit=3)
    ...
    store.put(("org", "acme", "research"), key=task_id, value={"summary": answer})
```

### 8. Comparison of approaches

| Approach | Pros | Cons |
|---|---|---|
| One shared `messages` list for all agents | Simplest | Context bloat, unclear attribution |
| Per-agent output keys | Clear ownership, no conflicts | Schema grows with agents |
| Namespaced dict + merge reducer | Scales to dynamic agents | Weaker typing |
| Task ledger | Progress tracking, retries, UI | Need a careful reducer (upsert) |
| Subgraph private state | Isolation, small parent state | Must design what crosses the boundary |

## Common Pitfalls

- Several agents writing the same non-reducer key in parallel, which raises `InvalidUpdateError`.
- An `operator.add` ledger, where status updates **append duplicates** instead of updating. Use an upsert reducer.
- Dumping every agent's raw tool messages into shared `messages`, which pollutes the other agents' context and costs tokens.
- Storing large artifacts (files, dataframes) in state. Store a reference (S3 key or ID) instead.
- No timestamps or agent names on events, which leaves you unable to reconstruct what happened.
- Relying on traces alone. Traces are for ops. Business-level progress belongs in state.

## Likely Follow-up Questions

1. **Shared state vs message passing?** LangGraph uses shared state with reducers (a blackboard). Message passing (AutoGen-style) means each agent sees a conversation. You can emulate either.
2. **How do you show live progress to users?** Stream `updates` or `custom` events with `subgraphs=True`, keyed by agent namespace.
3. **How do you resume a half-finished multi-agent job?** The checkpointer holds the ledger. Re-invoke with the same `thread_id`, and pending tasks continue.
4. **How do you avoid a giant state?** Private subgraph state, summaries, references to external storage, and trimming messages.
5. **How do you audit who approved what?** Write human decisions as ledger and events entries. They persist in checkpoint history and traces.

[← Back to questions](../questions.md)
