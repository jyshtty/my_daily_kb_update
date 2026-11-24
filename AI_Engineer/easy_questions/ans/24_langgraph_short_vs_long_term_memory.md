# Q24. Short-term vs long-term memory in LangGraph

[← Back to questions](../questions.md)

## TL;DR

- **Short-term memory** is **thread-scoped**: the conversation and working state of one session. It lives in graph **state** and is persisted by the **checkpointer**, keyed by `thread_id`.
- **Long-term memory** is **cross-thread**: facts, preferences, and learned instructions that survive across sessions. It lives in a **Store** (`InMemoryStore`, `PostgresStore`) and is organised by **namespaces** (tuples like `("users", user_id, "memories")`) plus keys.
- Stores support **semantic search** when configured with an embedding index: `InMemoryStore(index={"embed": ..., "dims": ...})` and then `store.search(namespace, query="...")`.
- Short-term memory grows without limit, so manage it with **trimming** (`trim_messages`), **deleting** (`RemoveMessage`), or **summarising** (a summary node, or `SummarizationMiddleware` in LangChain v1).
- Long-term memory types: **semantic** (facts), **episodic** (past experiences / few-shot examples), and **procedural** (instructions/prompts the agent updates). Write them **in the hot path** (during the turn) or **in the background** (asynchronously after the turn).

## Detailed Answer

### 1. The two scopes

```
 thread "t-1" (Mon)          thread "t-2" (Fri)
 ┌───────────────────┐        ┌───────────────────┐
 │ messages: [...]   │        │ messages: [...]   │   <- short-term (checkpointer)
 │ scratch state     │        │ scratch state     │      isolated per thread
 └─────────┬─────────┘        └─────────┬─────────┘
           │ put()                       │ search()
           ▼                             ▼
 ┌──────────────────────────────────────────────────┐
 │ Store: ("users","u-42","memories")               │   <- long-term (store)
 │   key=pref-1  {"text": "prefers vegetarian"}     │      shared across threads
 │   key=fact-7  {"text": "lives in Pune"}          │
 └──────────────────────────────────────────────────┘
```

| | Short-term | Long-term |
|---|---|---|
| Scope | One thread | Across threads (per user, org, app) |
| Storage | Checkpointer (`InMemorySaver`, `PostgresSaver`) | Store (`InMemoryStore`, `PostgresStore`, Redis...) |
| Keyed by | `thread_id` (+ checkpoint) | `namespace` tuple + `key` |
| Content | Messages, intermediate state | JSON documents (facts, profiles, instructions) |
| Retrieval | Automatic (state loaded on invoke) | Explicit: `get`, `search` (filter / semantic) |
| Written by | Node return values via reducers | Explicit `store.put` in nodes or tools, or background jobs |
| Growth issue | Context window overflow | Stale or contradictory memories |

### 2. Short-term memory

```python
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import InMemorySaver

def chat(state: MessagesState):
    return {"messages": [llm.invoke(state["messages"])]}

g = StateGraph(MessagesState)
g.add_node("chat", chat); g.add_edge(START, "chat"); g.add_edge("chat", END)
app = g.compile(checkpointer=InMemorySaver())

cfg = {"configurable": {"thread_id": "t-1"}}
app.invoke({"messages": [("user", "I'm Ajay")]}, cfg)
app.invoke({"messages": [("user", "Who am I?")]}, cfg)   # remembers within t-1
```

#### Managing context size

**a) Trim when calling the model.** State stays intact. Only the LLM input is shortened.

```python
from langchain_core.messages import trim_messages
from langchain_core.messages.utils import count_tokens_approximately

def chat(state: MessagesState):
    msgs = trim_messages(
        state["messages"],
        strategy="last",
        token_counter=count_tokens_approximately,
        max_tokens=2000,
        start_on="human",       # don't start on an orphan AI/tool message
        include_system=True,
    )
    return {"messages": [llm.invoke(msgs)]}
```

**b) Delete from state permanently** with `RemoveMessage`:

```python
from langchain_core.messages import RemoveMessage
def prune(state: MessagesState):
    return {"messages": [RemoveMessage(id=m.id) for m in state["messages"][:-10]]}
```

**c) Summarise** older turns into a running summary:

```python
class State(MessagesState):
    summary: str

def summarize(state: State):
    if len(state["messages"]) <= 12:
        return {}
    prompt = (f"Current summary: {state.get('summary', '')}\n"
              "Extend the summary with the conversation above.")
    new_summary = llm.invoke(state["messages"] + [("user", prompt)]).content
    return {"summary": new_summary,
            "messages": [RemoveMessage(id=m.id) for m in state["messages"][:-4]]}

def chat(state: State):
    sys = [("system", f"Conversation summary: {state['summary']}")] if state.get("summary") else []
    return {"messages": [llm.invoke(sys + state["messages"])]}
```

LangChain v1 agents can use `SummarizationMiddleware`, and `langmem` offers `SummarizationNode` for prebuilt agents.

Careful when trimming: never separate an `AIMessage` that has `tool_calls` from its matching `ToolMessage`s. Providers reject orphaned tool calls.

### 3. Long-term memory with the Store

#### Basic API

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
ns = ("users", "u-42", "memories")
store.put(ns, "pref-1", {"text": "Prefers vegetarian food"})
item = store.get(ns, "pref-1")            # Item(value=..., key, namespace, created_at, updated_at)
items = store.search(ns, filter={"text": "Prefers vegetarian food"}, limit=10)
store.delete(ns, "pref-1")
store.list_namespaces(prefix=("users",))
```

#### Semantic search

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(index={
    "embed": init_embeddings("openai:text-embedding-3-small"),
    "dims": 1536,
    "fields": ["text"],          # which JSON fields to embed
})
store.put(ns, "m1", {"text": "Allergic to peanuts"})
store.put(ns, "m2", {"text": "Works as a data engineer"})
hits = store.search(ns, query="what food should I avoid?", limit=2)
hits[0].value, hits[0].score
```

#### Using the store inside a graph

```python
import uuid
from langgraph.store.base import BaseStore
from langchain_core.runnables import RunnableConfig

def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    user_id = config["configurable"]["user_id"]
    ns = ("users", user_id, "memories")
    memories = store.search(ns, query=state["messages"][-1].content, limit=5)
    mem_text = "\n".join(f"- {m.value['text']}" for m in memories)

    response = llm.invoke([("system", f"Known about the user:\n{mem_text}")] + state["messages"])

    # naive hot-path write: store explicit "remember ..." requests
    last = state["messages"][-1].content
    if last.lower().startswith("remember"):
        store.put(ns, str(uuid.uuid4()), {"text": last[len("remember"):].strip()})
    return {"messages": [response]}

g = StateGraph(MessagesState)
g.add_node("call_model", call_model)
g.add_edge(START, "call_model")
app = g.compile(checkpointer=InMemorySaver(), store=store)

app.invoke({"messages": [("user", "Remember I'm allergic to peanuts")]},
           {"configurable": {"thread_id": "t-1", "user_id": "u-42"}})
app.invoke({"messages": [("user", "Suggest a snack")]},
           {"configurable": {"thread_id": "t-2", "user_id": "u-42"}})   # new thread, still knows
```

Production: `from langgraph.store.postgres import PostgresStore` (with `.setup()`, and pgvector for the index). On LangGraph Platform the store is provided automatically.

### 4. Kinds of long-term memory

| Type | What | Example | How stored |
|---|---|---|---|
| **Semantic** | Facts about user or world | "Lives in Pune" | Profile doc (one JSON) or a collection of facts |
| **Episodic** | Past experiences | "Last time, solution X worked" | Few-shot examples, past trajectories |
| **Procedural** | How to behave | Updated system prompt / rules | Instructions doc the agent rewrites |

Profile versus collection: a **profile** is one document that gets updated (easy to read, risk of overwrite conflicts). A **collection** is many small memories (easier to add to, but needs dedup and consolidation).

### 5. When to write memories

| Approach | Pros | Cons |
|---|---|---|
| **Hot path** (agent calls `save_memory` tool during the turn) | Immediately available, transparent | Adds latency, distracts the agent |
| **Background** (async job after conversation, e.g. LangMem memory manager) | No latency, can consolidate | Delayed, needs scheduling |

`langmem` provides `create_manage_memory_tool`, `create_search_memory_tool`, and background memory managers that work with the LangGraph store.

## Common Pitfalls

- Putting user facts only in the thread state and expecting them in a new thread. They have to go in the **store**.
- Hard-coding namespaces without `user_id`, which leaks memories between users.
- Trimming in a way that orphans tool calls, which causes provider errors.
- Storing everything. Memory stays useful only with relevance filtering, dedup, and updates for contradictions ("moved from Pune to Mumbai").
- Forgetting to pass `store=` at compile time. The `store` parameter then won't be injected.
- PII and compliance. Memories need deletion paths (`store.delete`, `delete_thread`).

## Likely Follow-up Questions

1. **Can the checkpointer act as long-term memory?** Only within a thread. For cross-thread memory use the store, or you'd have to query across threads yourself.
2. **How is the store injected into nodes?** Declare a `store: BaseStore` keyword parameter (or use `get_store()` / the `Runtime` object in newer versions). LangGraph injects the store passed to `compile`.
3. **Trim vs summarise?** Trimming is cheap but loses information. Summarising keeps the gist at the cost of an extra LLM call.
4. **How do you search memories semantically?** Configure the store `index` with embeddings and dims, then `store.search(ns, query=...)`.
5. **How do you keep memories consistent?** Background consolidation, upserts by key, timestamps, and having an LLM reconcile conflicting facts.

[← Back to questions](../questions.md)
