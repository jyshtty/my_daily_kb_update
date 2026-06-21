# LangGraph State

## MessagesState

`MessagesState` is a built-in LangGraph `TypedDict` with a single key:

```python
from langgraph.graph import MessagesState

# Equivalent to:
class MessagesState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
```

The `add_messages` reducer **appends** incoming messages instead of replacing the list. When a node returns `{"messages": [response]}`, LangGraph merges it into the existing conversation history.

## Custom State

You can define your own state by subclassing `TypedDict` with `Annotated` reducers:

```python
from typing import Annotated, TypedDict, Optional

class Logs(TypedDict):
    id: str
    question: str
    answer: str
    grade: Optional[int]
    feedback: Optional[str]

def add_logs(left: list[Logs], right: list[Logs]) -> list[Logs]:
    return left + right

class MyState(TypedDict):
    logs: Annotated[list[Logs], add_logs]  # append reducer
    report: str                             # overwrite (no reducer)
```

- Fields **with** a reducer: merged/accumulated on each update
- Fields **without** a reducer: overwritten on each update

## State Sharing in Subgraphs

For a parent graph to pass state into a subgraph, both must share the **same key name** with **compatible reducers**:

```python
class EntryGraphState(TypedDict):
    logs: Annotated[list[Logs], add_logs]  # matches subgraph key name

entry_builder.add_node("failure_analysis", failure_analysis_agent_builder.compile())
```

The subgraph reads `logs` from the parent state and writes its output keys (`failure_report`, `summary_report`) back up.

## Prebuilt Utilities

### `ToolNode`

A prebuilt node that executes tool calls. It reads `tool_calls` from the last message, runs each function, and appends the results as `ToolMessage`s to state.

```python
from langgraph.prebuilt import ToolNode

tool_node = ToolNode(tools)
workflow.add_node("tools", tool_node)
```

### `tools_condition`

A prebuilt conditional edge function. Routes to `"tools"` if the last message has `tool_calls`, otherwise routes to `END`.

```python
from langgraph.prebuilt import tools_condition

workflow.add_conditional_edges("agent", tools_condition)
```

Equivalent to writing this manually:

```python
def should_continue(state: MessagesState) -> Literal["tools", END]:
    last_message = state['messages'][-1]
    if last_message.tool_calls:
        return "tools"
    return END

workflow.add_conditional_edges("agent", should_continue)
```

## Reducers Summary

| Reducer | Behavior |
|---|---|
| `add_messages` | Appends messages, deduplicates by ID |
| Custom `add_*` | Any merge logic (append, union, sum, etc.) |
| None | Last write wins (overwrite) |
