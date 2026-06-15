# ReAct Loop (Reasoning + Acting)

## What is ReAct?

ReAct is a pattern where the LLM **reasons** and **acts** in a loop until it has a final answer. The name comes from **Re**asoning + **Act**ing.

## How the Loop Works

```
User Question
    → LLM thinks: "What do I need to do?"
    → calls a tool
    → Tool returns result
    → LLM thinks: "Do I have enough info?"
        YES → return final answer
        NO  → call another tool → repeat
```

## Real Example

```
User: "What files in the repo contain TODO comments?"

LLM thinks: "I need to search the repo"
    → calls search_docs_tool("TODO")

Tool returns results
    → LLM thinks: "Let me get the file content"
    → calls get_file_content_tool("main.py")

Tool returns content
    → LLM thinks: "I have enough info now"
    → returns final answer to user
```

## LangGraph ReAct Pattern

```python
# The loop in LangGraph
[LLM Node] → should I call a tool?
    YES → [ToolNode] → back to [LLM Node]
    NO  → END
```

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(llm, tools=[search_docs, get_file_content])
# LLM dynamically decides which tool to call and when
```

## Workflow vs ReAct Agent

| | Workflow (Fixed) | ReAct Agent (Dynamic) |
|---|---|---|
| **Flow** | Hardcoded: A → B → C | LLM decides next step |
| **Intelligence** | Deterministic | LLM-driven |
| **Tools** | Plain function nodes | ToolNode |
| **Example** | booking → housekeeping → customer_service | LLM picks tools based on query |

## Key Insight

- In a **workflow**, YOU decide the path via edges
- In a **ReAct agent**, the **LLM decides** the path by choosing which tools to call

The LLM keeps looping until it decides it has enough information to answer.

