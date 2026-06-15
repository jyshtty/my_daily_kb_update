## TavilySearch Tool

**`tool = TavilySearch(max_results=5)` — initializes the Tavily web search tool that returns up to 5 results. Commonly used in LangGraph agents to give LLMs real-time web search capability.**

---

### Installation

```bash
pip install langchain-tavily
```

Set your API key:

```bash
export TAVILY_API_KEY="your_tavily_api_key"
```

---

### Basic Usage

```python
from langchain_tavily import TavilySearch

tool = TavilySearch(max_results=5)

results = tool.invoke("What is LangGraph?")
print(results)
```

---

### Using TavilySearch in a LangGraph Agent

```python
from typing import TypedDict, List, Annotated
from langchain_tavily import TavilySearch
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
import operator


# State
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]


# Initialize Tavily tool — returns up to 5 search results
tool = TavilySearch(max_results=5)
tools = [tool]

# LLM bound with the tool
llm = ChatAnthropic(model="claude-sonnet-4-6")
llm_with_tools = llm.bind_tools(tools)


# Agent node — LLM decides whether to call the tool
def agent(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}


# Router — continue to tool call or end
def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return END


# Build graph
builder = StateGraph(AgentState)

builder.add_node("agent", agent)
builder.add_node("tools", ToolNode(tools))

builder.set_entry_point("agent")
builder.add_conditional_edges("agent", should_continue)
builder.add_edge("tools", "agent")

graph = builder.compile()

# Run
from langchain_core.messages import HumanMessage

result = graph.invoke({
    "messages": [HumanMessage(content="What are the latest updates on LangGraph in 2025?")]
})

print(result["messages"][-1].content)
```

---

### TavilySearch Result Structure

Each result returned by Tavily looks like:

```python
{
    "url": "https://example.com/article",
    "content": "LangGraph is a library by LangChain for building...",
    "score": 0.98
}
```

---

### max_results Comparison

```python
# Returns top 1 result — fast, less context
tool = TavilySearch(max_results=1)

# Returns top 5 results — balanced (recommended default)
tool = TavilySearch(max_results=5)

# Returns top 10 results — richer context, more tokens
tool = TavilySearch(max_results=10)
```

---

### Key Takeaway

- `TavilySearch(max_results=5)` gives the agent real-time web access with 5 search results per query.
- Works seamlessly with LangGraph's `ToolNode` — no manual tool parsing needed.
- Increase `max_results` for richer context, decrease for speed and lower token usage.
