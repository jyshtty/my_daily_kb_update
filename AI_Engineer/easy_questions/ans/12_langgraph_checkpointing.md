# Q12. What is checkpointing in LangGraph?

[← Back to questions](../questions.md)

## TL;DR

- **Checkpointing** is LangGraph's built-in persistence. After every **super-step**, the graph state is saved as a **checkpoint** by a **checkpointer** (`InMemorySaver`, `SqliteSaver`, `PostgresSaver`, ...).
- Checkpoints are grouped into **threads** (`config={"configurable": {"thread_id": ...}}`). A thread is one conversation or run history, and each checkpoint has its own `checkpoint_id`.
- Each checkpoint is exposed as a **`StateSnapshot`** with `values`, `next`, `config`, `metadata`, `created_at`, `parent_config`, `tasks` (and `interrupts`).
- It enables **short-term memory** (multi-turn chat), **human-in-the-loop** (interrupt/resume), **time travel** (replay or fork from old checkpoints), and **fault tolerance** (resume after a crash).
- **Pending writes** save the outputs of nodes that succeeded within a super-step that later failed, so on retry only the failed nodes re-run.

## Detailed Answer

### 1. Super-steps: when checkpoints are taken

LangGraph's runtime is based on Pregel/BSP (bulk synchronous parallel). Execution runs in **super-steps**:

1. Work out which nodes are triggered (their input channels changed).
2. Run all of them **in parallel**.
3. Apply their writes to the state channels using the reducers.
4. **Save a checkpoint.**

```
 thread_id = "user-42"
 ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐
 │ ckpt  -1  │──▶│ ckpt  0   │──▶│ ckpt  1   │──▶│ ckpt  2   │
 │ source=   │   │ source=   │   │ source=   │   │ source=   │
 │  input    │   │  loop     │   │  loop     │   │  loop     │
 │ next=     │   │ next=     │   │ next=     │   │ next=()   │
 │ (__start__)│  │ (agent,)  │   │ (tools,)  │   │  done     │
 └───────────┘   └───────────┘   └───────────┘   └───────────┘
      step -1         step 0          step 1          step 2
```

Parallel branches running in the same super-step (fan-out) share one checkpoint. Their writes are merged together.

### 2. Core concepts

| Concept | Meaning |
|---|---|
| **Checkpointer** | Object implementing `BaseCheckpointSaver` (`put`, `put_writes`, `get_tuple`, `list`) |
| **Thread** | Logical history identified by `thread_id` |
| **Checkpoint** | Snapshot of all channel values + versions at one super-step |
| **checkpoint_ns** | Namespace. Empty string for the root graph, set for subgraphs |
| **Pending writes** | Writes from tasks finished in an incomplete super-step |
| **Metadata** | `source` (`input`/`loop`/`update`/`fork`), `step`, `writes`, `parents` |

### 3. StateSnapshot fields

```python
snap = graph.get_state({"configurable": {"thread_id": "user-42"}})
snap.values          # dict: current state channel values
snap.next            # tuple of node names to run next; () means finished
snap.config          # {'configurable': {'thread_id', 'checkpoint_ns', 'checkpoint_id'}}
snap.metadata        # {'source': 'loop', 'step': 2, 'writes': {...}, 'parents': {}}
snap.created_at      # ISO timestamp
snap.parent_config   # config of previous checkpoint (None for first)
snap.tasks           # tuple of PregelTask(id, name, path, error, interrupts, state, result)
snap.interrupts      # pending Interrupt objects (newer versions)
```

### 4. Basic usage

```python
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import InMemorySaver
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

def chat(state: MessagesState):
    return {"messages": [llm.invoke(state["messages"])]}

builder = StateGraph(MessagesState)
builder.add_node("chat", chat)
builder.add_edge(START, "chat")
builder.add_edge("chat", END)

graph = builder.compile(checkpointer=InMemorySaver())

cfg = {"configurable": {"thread_id": "user-42"}}
graph.invoke({"messages": [("user", "Hi, I'm Ajay")]}, cfg)
out = graph.invoke({"messages": [("user", "What's my name?")]}, cfg)
print(out["messages"][-1].content)   # "Your name is Ajay." (history restored)

# Different thread means a blank slate
graph.invoke({"messages": [("user", "What's my name?")]},
             {"configurable": {"thread_id": "other"}})
```

### 5. Use cases

#### a) Short-term (thread) memory
Each `invoke` on the same thread loads the latest checkpoint, so the messages accumulate (via the `add_messages` reducer) without the caller re-sending history.

#### b) Human-in-the-loop
`interrupt()` and static breakpoints work because the paused state is checkpointed. `Command(resume=...)` reloads it.

#### c) Time travel
```python
history = list(graph.get_state_history(cfg))      # newest first
past = history[2]
graph.invoke(None, past.config)                    # replay from that checkpoint
fork_cfg = graph.update_state(past.config, {"messages": [("user", "alt question")]})
graph.invoke(None, fork_cfg)                       # new branch
```

#### d) Fault tolerance and pending writes

Suppose super-step 3 runs nodes `A`, `B`, and `C` in parallel. `A` and `B` succeed and `C` raises (timeout).

```
super-step 3:  A ✔  -> put_writes(task A)  ┐
               B ✔  -> put_writes(task B)  ├─ stored as pending writes
               C ✘  (exception)            ┘
```

When you re-invoke with `graph.invoke(None, cfg)`, LangGraph reloads the last checkpoint and the **pending writes** of A and B. It runs **only C**, then completes the super-step. This saves money on expensive LLM calls and avoids repeating side effects.

#### e) Debugging and auditing
`get_state_history` gives a complete step-by-step record of what each node wrote (`metadata["writes"]`).

### 6. Checkpointer options

| Checkpointer | Package | Use |
|---|---|---|
| `InMemorySaver` (`MemorySaver`) | `langgraph` | Tests, notebooks. Lost on restart |
| `SqliteSaver` / `AsyncSqliteSaver` | `langgraph-checkpoint-sqlite` | Local apps, single node |
| `PostgresSaver` / `AsyncPostgresSaver` | `langgraph-checkpoint-postgres` | Production |
| Redis, MongoDB, etc. | community/partner packages | Specific infrastructure |
| LangGraph Platform | managed | Persistence handled automatically |

### 7. Checkpointer vs Store

The checkpointer stores **per-thread** state. For memory **across** threads (user preferences, facts), use a **Store** (`InMemoryStore`, `PostgresStore`) passed as `compile(store=...)`. The two are complementary (see Q24).

## Common Pitfalls

- **Not passing `thread_id`.** With a checkpointer this raises an error, because the config must include `thread_id`.
- **Using `InMemorySaver` in production.** Data is lost on restart and not shared across workers.
- **Unbounded threads.** The message list grows forever. Trim or summarise, and set TTL/cleanup policies.
- **Storing unserialisable objects in state** (clients, file handles). Checkpoint serialisation fails.
- **Assuming one checkpoint per node.** Checkpoints are per **super-step**, and parallel nodes share one.
- **Compiling subgraphs with their own checkpointer by accident.** Usually you let the parent's checkpointer propagate.

## Likely Follow-up Questions

1. **What is a super-step?** One tick of the Pregel loop. All triggered nodes run in parallel, their writes are applied, then a checkpoint is saved.
2. **How do you resume after a crash?** `graph.invoke(None, {"configurable": {"thread_id": tid}})`. LangGraph picks up from the latest checkpoint plus its pending writes.
3. **Thread vs checkpoint?** A thread is a sequence (a linked list via `parent_config`) of checkpoints. A checkpoint is one snapshot.
4. **Can you inspect state without running?** Yes: `graph.get_state(cfg)` and `graph.get_state_history(cfg)`.
5. **What does `durability` mean?** Newer versions accept `durability="exit" | "async" | "sync"` on invoke/stream. It controls when checkpoints are persisted, trading performance against safety.

[← Back to questions](../questions.md)
