# Q28. What is time travel in LangGraph and how do you fork from a past checkpoint?

[← Back to questions](../questions.md)

## TL;DR

- **Time travel** means going back to **any previous checkpoint** in a thread to **inspect**, **replay**, or **fork** execution. It relies on the checkpointer storing every super-step.
- **Inspect:** `graph.get_state_history(config)` returns `StateSnapshot`s, newest first, each with a `config` containing its `checkpoint_id`.
- **Replay:** `graph.invoke(None, {"configurable": {"thread_id": t, "checkpoint_id": cid}})`. Steps **up to** that checkpoint are not recomputed. Execution continues from there and **re-runs the nodes after it**.
- **Fork:** `new_cfg = graph.update_state(past_snapshot.config, {...new values...}, as_node=...)` creates a **new checkpoint branching from the old one**. `graph.invoke(None, new_cfg)` then runs the alternative timeline. The original history stays intact.
- Uses: **debugging** non-deterministic agent failures, **exploring alternatives** ("what if the tool returned X?"), **correcting mistakes** (edit a bad tool call and continue), human-in-the-loop edits, and building eval datasets.

## Detailed Answer

### 1. Mental model

Checkpoints form a **tree** through `parent_config`. Normal execution adds to one branch, and forking starts a new branch from an older node.

```
thread "t1"

 c0 (input) ── c1 (agent) ── c2 (tools) ── c3 (agent: wrong answer)   <- original branch
                    │
                    └── c2' (update_state: edited tool call) ── c3' ── c4' (correct)   <- fork
```

Every checkpoint is addressable by `(thread_id, checkpoint_ns, checkpoint_id)`. The latest checkpoint for the thread (what `get_state(cfg)` returns without an id) becomes the head of whichever branch was written **most recently**.

### 2. Example setup

```python
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import InMemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

@tool
def get_price(ticker: str) -> str:
    """Get a stock price."""
    return {"AAPL": "190", "MSFT": "410"}.get(ticker, "unknown")

llm = ChatOpenAI(model="gpt-4o-mini").bind_tools([get_price])

def agent(s: MessagesState):
    return {"messages": [llm.invoke(s["messages"])]}

g = StateGraph(MessagesState)
g.add_node("agent", agent)
g.add_node("tools", ToolNode([get_price]))
g.add_edge(START, "agent")
g.add_conditional_edges("agent", tools_condition)
g.add_edge("tools", "agent")
app = g.compile(checkpointer=InMemorySaver())

cfg = {"configurable": {"thread_id": "t1"}}
app.invoke({"messages": [("user", "Price of Apple?")]}, cfg)
```

### 3. Inspect history

```python
history = list(app.get_state_history(cfg))       # newest -> oldest
for snap in history:
    print(
        snap.metadata["step"],
        snap.metadata["source"],                # input | loop | update | fork
        snap.next,                               # what would run next from here
        snap.config["configurable"]["checkpoint_id"],
        len(snap.values.get("messages", [])),
    )
# 3 loop ()          1ef...a  4
# 2 loop ('agent',)  1ef...9  3
# 1 loop ('tools',)  1ef...8  2    <- right after the LLM proposed the tool call
# 0 loop ('agent',)  1ef...7  1
# -1 input ('__start__',) 1ef...6 0
```

You can also filter: `app.get_state_history(cfg, filter={"source": "loop"}, limit=5, before=some_cfg)`.

### 4. Replay from a checkpoint

```python
before_tools = next(s for s in history if s.next == ("tools",))
for event in app.stream(None, before_tools.config, stream_mode="updates"):
    print(event)
```

- Passing `None` as input means "continue from this checkpoint".
- Nodes **before** the checkpoint are not re-executed, because their results are loaded from storage.
- Nodes **after** it **do run again**, with fresh LLM calls and fresh tool calls. Output can differ (non-determinism), and **side effects happen again**.
- This creates new checkpoints that branch from `before_tools`. It is effectively a fork with unchanged state.

### 5. Fork: edit the past, then continue

**Case: fix a wrong tool argument.**

```python
snap = before_tools
ai_msg = snap.values["messages"][-1]
fixed_calls = [{**ai_msg.tool_calls[0], "args": {"ticker": "MSFT"}}]
fixed_msg = ai_msg.model_copy(update={"tool_calls": fixed_calls})   # same id -> replace

fork_cfg = app.update_state(snap.config, {"messages": [fixed_msg]})
# fork_cfg has a NEW checkpoint_id whose parent is snap's checkpoint
print(app.get_state(fork_cfg).next)    # ('tools',)

result = app.invoke(None, fork_cfg)
print(result["messages"][-1].content)  # answer based on MSFT price
```

**Case: pretend a node produced a different output** with `as_node`:

```python
from langchain_core.messages import ToolMessage
fork_cfg = app.update_state(
    snap.config,
    {"messages": [ToolMessage(content="999", tool_call_id=ai_msg.tool_calls[0]["id"])]},
    as_node="tools",                    # as if 'tools' ran -> next is 'agent'
)
app.invoke(None, fork_cfg)
```

How `update_state` works:

- Values go through the state **reducers**, the same as a node's return value. That's why the same message `id` replaces instead of appending.
- `as_node` sets which node the update is attributed to. That determines `next`, based on that node's outgoing edges.
- The result is a **new checkpoint** with `metadata["source"] == "update"`, and the original checkpoint is untouched.

### 6. Time travel with interrupts and HITL

Time travel combines naturally with human review. An operator looks at a finished run in LangGraph Studio, picks the step where the agent went wrong, edits state, and re-runs from there. Studio's "fork" and "replay" buttons call these same APIs.

### 7. Use cases

| Use case | Technique |
|---|---|
| Debug "why did the agent do X?" | `get_state_history` and inspect `metadata["writes"]` per step |
| Reproduce a flaky failure | Replay from the checkpoint before the failing node |
| Explore alternatives | Fork with different tool outputs or user inputs |
| Correct a bad action | Edit the AI tool call and fork |
| Undo the last turn in a chat UI | Resume from the checkpoint before the last user message |
| Build eval datasets | Export states at decision points as test inputs |

### 8. Replay vs fork

| | Replay | Fork |
|---|---|---|
| API | `invoke(None, past_cfg)` | `update_state(past_cfg, values)` then `invoke(None, new_cfg)` |
| State changed? | No | Yes |
| New checkpoints? | Yes (new branch, same starting state) | Yes (new branch from modified state) |
| Typical goal | Reproduce or debug | What-if, correction |

## Common Pitfalls

- **Assuming replay is deterministic.** LLM calls re-run. Use `temperature=0` or cached or mocked models when you need reproducibility.
- **Side effects re-executing** (emails, payments) when replaying past nodes. Guard them with idempotency keys or disable them in debug mode.
- **Appending instead of replacing** when editing messages. Reuse the message `id`.
- **Orphan tool calls** when forking. Every `tool_call_id` still needs a `ToolMessage`.
- **Forgetting `checkpoint_ns` for subgraphs.** Use `get_state(cfg, subgraphs=True)` to get subgraph configs.
- **`InMemorySaver`** loses history on restart, so production debugging needs a durable saver.

## Likely Follow-up Questions

1. **Does forking delete the original branch?** No. Checkpoints are immutable, and both branches remain in history.
2. **How do you go back to the original branch?** Invoke with that branch's latest `checkpoint_id`.
3. **What does `as_node` do in `update_state`?** It attributes the update to a node, which determines the next node(s) through that node's edges.
4. **Can you time-travel without a checkpointer?** No. Time travel needs persisted checkpoints.
5. **How do you implement "regenerate last answer"?** Find the checkpoint just before the final agent step and `invoke(None, that_config)`.

[← Back to questions](../questions.md)
