# Agents Frameworks Overview

## Tool Use with LLMs (OpenAI Function Calling)

LLMs can be given tools (functions) via the `tools` parameter. The model decides when to call them — it doesn't execute them, it returns a `tool_calls` object with the function name and arguments. The developer then calls the function and feeds the result back into the conversation.

```python
tools = [{"type": "function", "function": {"name": "my_fn", "description": "...", "parameters": {...}}}]
response = client.chat.completions.create(model=..., messages=messages, tools=tools)

if response.choices[0].message.tool_calls:
    args = json.loads(response.choices[0].message.tool_calls[0].function.arguments)
    result = my_fn(**args)
    messages.append({"role": "tool", "content": json.dumps(result), "tool_call_id": ...})
```

## LangChain Agents

LangChain wraps the tool-calling loop with syntactic sugar. Tools are defined with `@tool`, bound to the model with `model.bind_tools(tools)`, and an `AgentExecutor` handles the loop.

```python
from langchain_core.tools import tool
from langchain.agents import create_tool_calling_agent, AgentExecutor

@tool()
def my_fn(question: str) -> str:
    """Describe what this tool does."""
    return "result"

agent = create_tool_calling_agent(model, [my_fn], prompt)
executor = AgentExecutor(agent=agent, tools=[my_fn])
result = executor.invoke({"input": "user question"})
```

## LangGraph Agents

LangGraph builds stateful agents as graphs. Nodes are functions, edges define flow, and `MemorySaver` adds persistence. The ReAct loop is expressed as conditional edges: if tool calls exist → route to ToolNode, else → END.

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode

workflow = StateGraph(MessagesState)
workflow.add_node("agent", call_model)
workflow.add_node("tools", ToolNode(tools))
workflow.add_edge(START, "agent")
workflow.add_conditional_edges("agent", should_continue)  # routes to "tools" or END
workflow.add_edge("tools", "agent")
app = workflow.compile(checkpointer=MemorySaver())
```

### Multi-Agent with LangGraph Subgraphs

Compiled graphs can be used as nodes inside a parent graph, composing complex multi-agent workflows from smaller specialized graphs.

```python
entry_builder.add_node("failure_analysis", failure_analysis_agent_builder.compile())
entry_builder.add_node("question_summarization", question_summary_agent_builder.compile())
```

State sharing between subgraphs requires matching key names and compatible reducer functions (e.g., `Annotated[list[Logs], add_logs]`).

## CrewAI Agents

CrewAI organizes agents into a **Crew** with defined roles and tasks. Key concepts:

| Concept | Description |
|---|---|
| **Agent** | Autonomous unit with a role, goal, and backstory |
| **Task** | A job assigned to an agent with expected output |
| **Tool** | Decorated function or built-in CrewAI tool |
| **Crew** | Group of agents + tasks with a process (sequential/parallel) |
| **Pipeline** | Structured multi-crew workflow |

```python
from crewai import Agent, Task, Crew, Process

agent = Agent(role="Researcher", goal="Find info", backstory="...", llm=model)
task = Task(description="Research X", agent=agent, expected_output="Summary")
crew = Crew(agents=[agent], tasks=[task], process=Process.sequential)
result = crew.kickoff()
```

## Framework Comparison

| | OpenAI Function Calling | LangChain Agent | LangGraph | CrewAI |
|---|---|---|---|---|
| **Abstraction** | Low | Medium | Low-Medium | High |
| **Multi-agent** | Manual | Via chaining | Native (subgraphs) | Native (Crew) |
| **State/Memory** | Manual | ConversationMemory | Built-in (MemorySaver) | Built-in |
| **Control flow** | Manual loop | AgentExecutor | Graph edges | Process type |
| **Best for** | Simple tool calls | Quick agents | Complex stateful flows | Role-based teams |
