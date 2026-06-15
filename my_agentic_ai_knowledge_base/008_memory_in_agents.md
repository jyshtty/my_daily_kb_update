# Memory in Agents

## Types of Memory

### 1. Short-Term Memory (In-Context)
- Lives inside the **current conversation state**
- Lost when the session ends
- Fast, no external dependency

```python
class AgentState(TypedDict):
    messages: list[BaseMessage]  # this IS the short-term memory
```

### 2. Long-Term Memory (External Store)
- Persists across sessions — stored in a database, vector store, or file
- Must be explicitly written and retrieved
- Examples: user preferences, past interactions, facts

```python
# Write to long-term memory
vectorstore.add_texts(["User prefers formal tone"])

# Retrieve from long-term memory
docs = vectorstore.similarity_search("user preferences")
```

## Four Memory Categories

| Category | What it stores | Example |
|---|---|---|
| **Episodic** | Past events / conversations | "Last time user asked about X" |
| **Semantic** | Facts and knowledge | "Paris is the capital of France" |
| **Procedural** | How to do things | System prompt, tool instructions |
| **Working** | Current task context | Messages in state |

## LangGraph Memory Pattern

```python
# Short-term: state passed between nodes
def agent_node(state: AgentState):
    messages = state["messages"]  # read memory
    ...
    return {"messages": [new_message]}  # write memory

# Long-term: explicit retrieval step
def retrieve_memory(state: AgentState):
    query = state["messages"][-1].content
    docs = vectorstore.similarity_search(query)
    return {"context": docs}
```

## Key Rule
- Use **short-term** for within-session context
- Use **long-term** for cross-session knowledge

