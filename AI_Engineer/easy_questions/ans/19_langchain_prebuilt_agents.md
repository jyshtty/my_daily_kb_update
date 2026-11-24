# Q19. What prebuilt agent types are available in LangChain?

[← Back to questions](../questions.md)

## TL;DR

- **Current (LangChain v1)**: one main constructor, **`langchain.agents.create_agent`**. It is a tool-calling (ReAct-style) agent built on LangGraph, customised with **middleware** (`HumanInTheLoopMiddleware`, `SummarizationMiddleware`, PII redaction, model/tool call limits, and more) and `response_format` for structured output.
- **LangGraph prebuilt**: `langgraph.prebuilt.create_react_agent`, plus `ToolNode` and `tools_condition`. It is widely used, and LangChain v1 now points users to `create_agent` instead. Multi-agent helpers live in `langgraph-supervisor` (`create_supervisor`) and `langgraph-swarm` (`create_swarm`).
- **Legacy (`AgentExecutor` era, now in `langchain-classic` for v1)**: `create_tool_calling_agent`, `create_openai_tools_agent`, `create_openai_functions_agent`, `create_react_agent` (text ReAct), `create_structured_chat_agent`, `create_json_chat_agent`, `create_xml_agent`, `create_self_ask_with_search_agent`, all run via `AgentExecutor`.
- **Oldest (deprecated)**: `initialize_agent(tools, llm, agent=AgentType.X)` with types such as `ZERO_SHOT_REACT_DESCRIPTION`, `CONVERSATIONAL_REACT_DESCRIPTION`, `STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION`, `OPENAI_FUNCTIONS`, `SELF_ASK_WITH_SEARCH`, `REACT_DOCSTORE`.
- **Toolkit agents**: `create_sql_agent`, `create_pandas_dataframe_agent`, `create_csv_agent`, `create_json_agent`, `create_openapi_agent`, `create_spark_dataframe_agent` (community/experimental).

## Detailed Answer

### 1. Evolution

```
2022-23  initialize_agent + AgentType enum          (deprecated)
2023-24  create_*_agent  + AgentExecutor           (legacy -> langchain-classic)
2024-25  langgraph.prebuilt.create_react_agent      (LangGraph runtime)
2025+    langchain.agents.create_agent + middleware (LangChain v1, built on LangGraph)
```

The reason for the change: `AgentExecutor` was a black-box loop, hard to customise and without persistence or HITL. Rebuilding agents on LangGraph added checkpointing, streaming, interrupts, and composability.

### 2. LangChain v1: `create_agent`

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware, SummarizationMiddleware
from langchain_core.tools import tool
from langgraph.checkpoint.memory import InMemorySaver
from pydantic import BaseModel

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return f"Sunny in {city}"

class Answer(BaseModel):
    city: str
    summary: str

agent = create_agent(
    model="openai:gpt-4o",                 # or a ChatModel instance
    tools=[get_weather],
    system_prompt="You are a helpful weather assistant.",
    response_format=Answer,                # structured final output
    middleware=[
        SummarizationMiddleware(model="openai:gpt-4o-mini", max_tokens_before_summary=4000),
        HumanInTheLoopMiddleware(interrupt_on={"get_weather": False}),
    ],
    checkpointer=InMemorySaver(),
)

res = agent.invoke({"messages": [{"role": "user", "content": "Weather in Pune?"}]},
                   {"configurable": {"thread_id": "1"}})
res["structured_response"]   # Answer(city='Pune', summary='Sunny')
```

Middleware hooks (`before_model`, `after_model`, `wrap_model_call`, `wrap_tool_call`, ...) replace the many separate agent classes. Behaviour is composed instead of picking an agent type. Exact parameter names for individual middleware vary by version, so check the docs.

### 3. LangGraph prebuilt

```python
from langgraph.prebuilt import create_react_agent, ToolNode, tools_condition

agent = create_react_agent(
    model="openai:gpt-4o",
    tools=[get_weather],
    prompt="You are concise.",
    checkpointer=InMemorySaver(),
    # store=..., response_format=..., pre_model_hook=..., post_model_hook=...,
    # interrupt_before=["tools"], name="weather_agent"
)
```

`ToolNode` (runs tool calls from the last AI message, in parallel) and `tools_condition` (routes to `"tools"` or END) are building blocks for hand-rolled ReAct graphs.

Multi-agent prebuilts:

| Package | Function | Pattern |
|---|---|---|
| `langgraph-supervisor` | `create_supervisor(agents, model, prompt, output_mode=...)` | Central supervisor with handoff tools |
| `langgraph-swarm` | `create_swarm(agents, default_active_agent=...)`, `create_handoff_tool` | Peer handoffs |

### 4. Legacy constructors + `AgentExecutor`

```python
# LEGACY: in LangChain v1, import from langchain_classic.agents
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True,
                         max_iterations=10, handle_parsing_errors=True)
executor.invoke({"input": "Weather in Pune?"})
```

| Constructor | Mechanism | Notes |
|---|---|---|
| `create_tool_calling_agent` | Native tool calling (any provider supporting `bind_tools`) | Was the recommended legacy choice |
| `create_openai_tools_agent` | OpenAI `tools` API, parallel calls | OpenAI-specific |
| `create_openai_functions_agent` | Older OpenAI `functions` API (single call) | Superseded by tools |
| `create_react_agent` (langchain) | Text ReAct: `Thought/Action/Action Input/Observation`, parsed from text | Works with non-tool-calling LLMs. Brittle parsing. Name clashes with LangGraph's |
| `create_structured_chat_agent` | JSON blob actions, multi-input tools | For chat models without native tools |
| `create_json_chat_agent` | JSON-formatted actions | Similar |
| `create_xml_agent` | XML tags for actions | Suited to older Anthropic models |
| `create_self_ask_with_search_agent` | Self-ask decomposition with one search tool | Niche |

### 5. Oldest: `initialize_agent` and `AgentType` (deprecated)

```python
# DEPRECATED
from langchain.agents import initialize_agent, AgentType
agent = initialize_agent(tools, llm, agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION, verbose=True)
agent.run("What is 3 * 7?")
```

| `AgentType` | Description |
|---|---|
| `ZERO_SHOT_REACT_DESCRIPTION` | ReAct. Picks tools from their descriptions. No memory |
| `CHAT_ZERO_SHOT_REACT_DESCRIPTION` | Same, prompt tuned for chat models |
| `CONVERSATIONAL_REACT_DESCRIPTION` | ReAct + conversation memory |
| `CHAT_CONVERSATIONAL_REACT_DESCRIPTION` | Chat-model version of the above |
| `STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION` | Multi-input tools via JSON |
| `OPENAI_FUNCTIONS` | OpenAI function calling |
| `OPENAI_MULTI_FUNCTIONS` | Multiple function calls per step |
| `SELF_ASK_WITH_SEARCH` | Self-ask with an `Intermediate Answer` search tool |
| `REACT_DOCSTORE` | ReAct over a docstore with `Search` + `Lookup` tools |

### 6. Toolkit / specialised agents

```python
# SQL agent (langchain_community)
from langchain_community.utilities import SQLDatabase
from langchain_community.agent_toolkits import create_sql_agent
db = SQLDatabase.from_uri("sqlite:///chinook.db")
sql_agent = create_sql_agent(llm, db=db, agent_type="tool-calling", verbose=True)
sql_agent.invoke({"input": "Top 5 artists by sales?"})

# Pandas agent (langchain_experimental) - executes generated Python!
from langchain_experimental.agents import create_pandas_dataframe_agent
pd_agent = create_pandas_dataframe_agent(llm, df, agent_type="tool-calling",
                                         allow_dangerous_code=True)
pd_agent.invoke({"input": "Average revenue by region?"})
```

| Toolkit agent | Package | Purpose |
|---|---|---|
| `create_sql_agent` | `langchain_community.agent_toolkits` | NL → SQL with schema inspection and query checking |
| `create_pandas_dataframe_agent` | `langchain_experimental.agents` | Python REPL over a DataFrame |
| `create_csv_agent` | `langchain_experimental.agents` | Pandas agent over CSV files |
| `create_spark_dataframe_agent` | `langchain_experimental` | Spark DataFrames |
| `create_json_agent` | `langchain_community.agent_toolkits` | Explore large JSON |
| `create_openapi_agent` | `langchain_community.agent_toolkits` | Call APIs from an OpenAPI spec |
| `create_vectorstore_agent` | `langchain_community.agent_toolkits` | QA over vector stores |

Modern alternative: wrap the toolkit's tools (`SQLDatabaseToolkit(db=db, llm=llm).get_tools()`) and pass them to `create_agent` / `create_react_agent`.

### 7. Which to pick today

| Situation | Choice |
|---|---|
| New LangChain app | `langchain.agents.create_agent` |
| Need custom control flow | Hand-built LangGraph `StateGraph` with `ToolNode` |
| Existing LangGraph code | `langgraph.prebuilt.create_react_agent` (works, consider migrating) |
| Multiple agents | `create_supervisor` / `create_swarm` or a custom graph |
| Maintaining old code | `AgentExecutor` + `create_tool_calling_agent` (`langchain-classic`) |
| Model without tool calling | Text ReAct (legacy) or a custom parser |

## Common Pitfalls

- Confusing **`langchain.agents.create_react_agent`** (legacy text ReAct that needs a prompt with `{tools}`, `{tool_names}`, `{agent_scratchpad}`) with **`langgraph.prebuilt.create_react_agent`** (tool-calling graph).
- Using `initialize_agent` in new code. It is deprecated, with weak error handling.
- Pandas/Python agents run **arbitrary code**. Sandbox them, and `allow_dangerous_code=True` is required on purpose.
- SQL agents with write permissions. Use read-only DB users.
- Expecting `AgentExecutor` to persist memory or support HITL. It doesn't natively, and that is what LangGraph adds.

## Likely Follow-up Questions

1. **Why did LangChain move agents onto LangGraph?** For durability (checkpoints), HITL, streaming, and custom control flow. The opaque `AgentExecutor` loop could not support these well.
2. **What replaced the many agent types?** A single tool-calling agent plus middleware for customisation.
3. **How do you add memory to `create_agent`?** Pass `checkpointer=` and a `thread_id`, and add `store=` for long-term memory.
4. **How do you get structured output from an agent?** `response_format=PydanticModel`. The result is in `result["structured_response"]`.
5. **How do you limit runaway agents?** `recursion_limit` in config, call-limit middleware in v1, `max_iterations` in legacy `AgentExecutor`.

[← Back to questions](../questions.md)
