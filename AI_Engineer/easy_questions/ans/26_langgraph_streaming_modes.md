# Q26. What streaming modes does LangGraph support?

[← Back to questions](../questions.md)

## TL;DR

- `graph.stream(input, config, stream_mode=...)` (or `astream`) supports:
  - **`"values"`**: the **full state** after each super-step.
  - **`"updates"`**: only the **delta** each node returned, as `{node_name: update}`.
  - **`"messages"`**: **LLM tokens** as `(message_chunk, metadata)` tuples, from any chat model called inside nodes.
  - **`"custom"`**: arbitrary data you emit with `get_stream_writer()` (progress bars, tool progress).
  - **`"debug"`**: detailed execution events (tasks, results, checkpoints).
  - Also `"checkpoints"` and `"tasks"` in newer versions (subsets of debug).
- Pass a **list** (`stream_mode=["updates", "messages"]`) to get `(mode, chunk)` tuples. Pass **`subgraphs=True`** to include nested graph events with a **namespace**.
- Interrupts appear in the stream as `{"__interrupt__": (...)}` (in `updates`).
- `astream_events(version="v2")` is the older, LangChain-wide event stream. It is still available but heavier.

## Detailed Answer

### 1. Why streaming matters

Agents can take 5-60 seconds. Streaming improves **perceived latency** (tokens appear right away), **transparency** (showing "Searching the web..."), and **control** (a UI can render tool calls, approvals, partial results).

### 2. Example graph

```python
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.config import get_stream_writer
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

class State(MessagesState):
    docs: list[str]

def retrieve(state: State):
    writer = get_stream_writer()
    writer({"progress": "searching index..."})          # -> "custom" stream
    docs = ["doc A", "doc B"]
    writer({"progress": f"found {len(docs)} docs"})
    return {"docs": docs}

def answer(state: State):
    msg = llm.invoke([("system", f"Context: {state['docs']}")] + state["messages"])
    return {"messages": [msg]}

g = StateGraph(State)
g.add_node(retrieve); g.add_node(answer)
g.add_edge(START, "retrieve"); g.add_edge("retrieve", "answer"); g.add_edge("answer", END)
app = g.compile()
inp = {"messages": [("user", "What is in the docs?")], "docs": []}
```

### 3. Each mode

#### `values`: full state after every step

```python
for state in app.stream(inp, stream_mode="values"):
    print(list(state.keys()), len(state["messages"]))
# ['messages', 'docs'] 1     <- input
# ['messages', 'docs'] 1     <- after retrieve (docs filled)
# ['messages', 'docs'] 2     <- after answer
```

Good for "show the current state" UIs and simple scripts. It is heavy when the state is large.

#### `updates`: per-node deltas (the default for `stream`)

```python
for chunk in app.stream(inp, stream_mode="updates"):
    print(chunk)
# {'retrieve': {'docs': ['doc A', 'doc B']}}
# {'answer': {'messages': [AIMessage(...)]}}
```

Good for progress UIs ("node X finished") and for **interrupts**:

```python
# {'__interrupt__': (Interrupt(value={...}, id='...'),)}
```

#### `messages`: LLM tokens

```python
for chunk, meta in app.stream(inp, stream_mode="messages"):
    if chunk.content:
        print(chunk.content, end="", flush=True)
    # meta: {'langgraph_node': 'answer', 'langgraph_step': 2, 'ls_model_name': ..., 'tags': [...]}
```

- This works even though the node calls `llm.invoke` (not `.stream`). LangGraph hooks into LangChain's callback system to capture tokens.
- Filter by node with `meta["langgraph_node"] == "answer"`, or by **tags**: `llm.with_config(tags=["final"])` then check `"final" in meta["tags"]`.
- For a **non-LangChain** LLM client, use `custom` mode to emit tokens yourself.
- Tool-call chunks show up as `AIMessageChunk` with `tool_call_chunks`.

#### `custom`: your own events

```python
for chunk in app.stream(inp, stream_mode="custom"):
    print(chunk)          # {'progress': 'searching index...'} ...
```

You can call `get_stream_writer()` inside nodes **and tools**. On Python < 3.11 with async code, accept a `writer` parameter in the node instead, because context variables don't propagate there.

#### `debug`: everything

```python
for ev in app.stream(inp, stream_mode="debug"):
    print(ev["type"], ev["step"])    # 'checkpoint' | 'task' | 'task_result'
```

Use it for troubleshooting. It is too verbose for UIs.

### 4. Combining modes

```python
for mode, chunk in app.stream(inp, stream_mode=["updates", "messages", "custom"]):
    if mode == "messages":
        token, meta = chunk
        ui.append_token(token.content)
    elif mode == "custom":
        ui.show_status(chunk["progress"])
    elif mode == "updates":
        if "__interrupt__" in chunk:
            ui.ask_approval(chunk["__interrupt__"][0].value)
        else:
            ui.mark_done(next(iter(chunk)))
```

### 5. Subgraphs

```python
for namespace, chunk in app.stream(inp, stream_mode="updates", subgraphs=True):
    # namespace = () for root, ('research_agent:<task_id>',) for nested
    print(namespace, chunk)

# with multiple modes + subgraphs: (namespace, mode, chunk)
for ns, mode, chunk in app.stream(inp, stream_mode=["updates", "messages"], subgraphs=True):
    ...
```

This is essential for multi-agent UIs, so you can show which agent is speaking.

### 6. Async and web streaming (FastAPI SSE)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

api = FastAPI()

@api.post("/chat")
async def chat(body: dict):
    cfg = {"configurable": {"thread_id": body["thread_id"]}}
    async def gen():
        async for token, meta in app.astream(
            {"messages": [("user", body["text"])]}, cfg, stream_mode="messages"
        ):
            if token.content and meta["langgraph_node"] == "answer":
                yield f"data: {json.dumps({'token': token.content})}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")
```

On LangGraph Platform: `client.runs.stream(thread_id, assistant_id, input=..., stream_mode=["updates", "messages-tuple"])`. The SDK has its own mode names, such as `messages-tuple` and `events`.

### 7. Summary table

| Mode | Yields | Size | Typical use |
|---|---|---|---|
| `values` | Full state after each step | Large | Simple apps, state viewers |
| `updates` | `{node: delta}` | Small | Progress, interrupts, logs |
| `messages` | `(AIMessageChunk, metadata)` | Token-level | Chat UIs (typing effect) |
| `custom` | Anything via `get_stream_writer()` | Your choice | Tool progress, non-LangChain LLM tokens |
| `debug` | Task, result, and checkpoint events | Very large | Debugging |
| `checkpoints` / `tasks` | Checkpoint / task events | Medium | Observability tooling (newer versions) |
| `astream_events(v2)` | All LangChain callback events | Very large | Legacy, fine-grained events |

### 8. `stream` vs `invoke`

`invoke` is internally `stream(stream_mode="values")` keeping only the last chunk. Both run the same graph and checkpoint the same way, so choosing streaming has no effect on persistence or HITL.

## Common Pitfalls

- **Expecting tokens from `updates`.** Updates arrive only when a node **finishes**. Use `messages` for tokens.
- **Not filtering `messages` by node.** Tokens from internal LLM calls (router, summariser) leak to the user. Filter on `langgraph_node` or tags.
- **Forgetting `subgraphs=True`.** Nested agent events are invisible otherwise.
- **Python < 3.11 async + `get_stream_writer()`.** Context isn't propagated, so pass `writer` as a node argument.
- **Streaming `values` with big state** wastes bandwidth. Prefer `updates`.
- **Treating `__interrupt__` as the end of the stream with no follow-up.** The client must resume with `Command(resume=...)`.

## Likely Follow-up Questions

1. **How do you stream tokens if the node uses `invoke()`?** `stream_mode="messages"` captures them through callbacks anyway.
2. **How do you stream from a tool?** Call `get_stream_writer()` inside the tool and consume with `custom`.
3. **How do you tell which agent produced a token?** Use `metadata["langgraph_node"]` and the namespace when `subgraphs=True`.
4. **Can you stream multiple modes at once?** Yes, with a list of modes. The result is `(mode, chunk)` tuples.
5. **How does streaming interact with interrupts?** The stream yields `__interrupt__` and ends. Resume with a new `stream(Command(resume=...), cfg)` call.

[← Back to questions](../questions.md)
