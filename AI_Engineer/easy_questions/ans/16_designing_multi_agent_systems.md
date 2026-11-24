# Q16. How would you design and coordinate multiple agents using LangGraph or another orchestration framework?

[← Back to questions](../questions.md)

## TL;DR

- Start from the **task decomposition**. Split work into agents only where **specialisation** (different prompts, tools, models, or permissions) or **parallelism** pays off. Otherwise use one agent with good tools.
- Pick a **coordination pattern**: **sequential pipeline**, **supervisor** (central router), **hierarchical** (supervisors of supervisors), **swarm/handoffs** (agents transfer control directly), or **network** (any-to-any).
- In LangGraph, each agent is a **node or subgraph**. Coordination happens through **edges, conditional routing, `Command(goto=...)`, and `Send`**. Context is shared via **state** (shared or private keys), and a checkpointer makes it durable.
- Other frameworks: **CrewAI** (role/task crews, sequential or hierarchical processes), **AutoGen / AG2** (conversational group chats), **OpenAI Agents SDK** (agents + handoffs + guardrails), **Google ADK**, **Semantic Kernel**.
- Design for **observability, bounded loops, structured hand-off contracts, evaluation, and cost**. Multi-agent systems increase tokens and latency, so justify them.

## Detailed Answer

### 1. When to go multi-agent (and when not)

**Good reasons**

- Tool overload: more than about 10-15 tools in one agent degrades tool selection.
- Distinct expertise or prompts (legal reviewer vs code writer).
- Different security boundaries (only one agent can write to production).
- Parallel independent subtasks (research 5 companies at once).
- Different models per step (a cheap classifier, an expensive reasoner).

**Reasons NOT to**

- The task is a linear workflow. A plain graph or chain is cheaper and more reliable.
- Agents would need **all** of the same context anyway. Splitting only adds hand-off loss.
- Latency or cost budgets are tight. Every hop adds LLM calls.
- You can't yet evaluate a single agent well. More agents mean more failure modes.

Rule of thumb: **workflow first, single agent second, multi-agent last.**

### 2. Coordination patterns

```
Sequential pipeline      Supervisor               Hierarchical
A -> B -> C -> D            [Sup]                    [Top]
                           / | \                    /     \
                          A  B  C              [SupX]     [SupY]
                                                /  \       /  \
                                               A    B     C    D

Swarm / handoffs          Network (any-to-any)     Map-reduce (parallel)
A <-> B                   A <-> B                  plan -> Send(worker)*N -> reduce
 \   /                    | \ / |
   C                      C <-> D
```

| Pattern | Control | Pros | Cons | Fits |
|---|---|---|---|---|
| Sequential | Fixed edges | Predictable, cheap | Inflexible | ETL-like flows, content pipelines |
| Supervisor | LLM router | Flexible, central policy | Router bottleneck, extra hops | Assistants with specialist workers |
| Hierarchical | Nested supervisors | Scales to many agents | Complexity, latency | Large orgs of agents |
| Swarm/handoff | Active agent decides | Fewer hops, natural conversations | Harder to enforce global rules | Customer support tiers |
| Network | Any agent | Maximum flexibility | Loops, hard to debug | Research, rarely in prod |
| Map-reduce | `Send` fan-out | Parallel speed | Merge logic needed | Batch research, doc processing |

### 3. LangGraph building blocks

| Need | LangGraph primitive |
|---|---|
| Agent | Node function, `create_react_agent`, `create_agent`, or a compiled subgraph |
| Fixed order | `add_edge` |
| Decision | `add_conditional_edges(node, router_fn)` |
| Agent picks next + updates state | return `Command(goto="x", update={...})` |
| Jump to parent graph | `Command(goto="x", graph=Command.PARENT)` |
| Fan-out | conditional edge returning `[Send("worker", payload), ...]` |
| Shared memory | State keys with reducers |
| Private memory | Subgraph with its own state schema |
| Durability / HITL | checkpointer + `interrupt()` |
| Long-term memory | `store=InMemoryStore()` / `PostgresStore` |

### 4. Supervisor in LangGraph (custom)

```python
from typing import Literal
from pydantic import BaseModel
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.types import Command
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o")
research_agent = create_react_agent(llm, tools=[web_search], name="researcher",
                                    prompt="You research facts. Cite sources.")
coder_agent = create_react_agent(llm, tools=[python_repl], name="coder",
                                 prompt="You write and run Python.")

class Route(BaseModel):
    next: Literal["researcher", "coder", "FINISH"]

def supervisor(state: MessagesState) -> Command[Literal["researcher", "coder", "__end__"]]:
    decision = llm.with_structured_output(Route).invoke(
        [("system", "Route to researcher or coder; FINISH when the task is done.")]
        + state["messages"]
    )
    return Command(goto=END if decision.next == "FINISH" else decision.next)

def make_worker(agent, name):
    def node(state: MessagesState) -> Command[Literal["supervisor"]]:
        result = agent.invoke(state)
        # return only the final answer to keep shared context lean
        return Command(goto="supervisor",
                       update={"messages": [("ai", f"[{name}] {result['messages'][-1].content}")]})
    return node

g = StateGraph(MessagesState)
g.add_node("supervisor", supervisor)
g.add_node("researcher", make_worker(research_agent, "researcher"))
g.add_node("coder", make_worker(coder_agent, "coder"))
g.add_edge(START, "supervisor")
app = g.compile()
```

### 5. Prebuilt libraries

```python
# Supervisor  (pip install langgraph-supervisor)
from langgraph_supervisor import create_supervisor
workflow = create_supervisor(
    agents=[research_agent, coder_agent],
    model=llm,
    prompt="You manage a researcher and a coder.",
    output_mode="last_message",       # or "full_history"
)
app = workflow.compile(checkpointer=InMemorySaver())

# Swarm  (pip install langgraph-swarm)
from langgraph_swarm import create_swarm, create_handoff_tool
alice = create_react_agent(llm, [add, create_handoff_tool(agent_name="Bob")],
                           prompt="You are Alice, a math expert.", name="Alice")
bob = create_react_agent(llm, [create_handoff_tool(agent_name="Alice")],
                         prompt="You are Bob, a pirate-speaking assistant.", name="Bob")
swarm = create_swarm([alice, bob], default_active_agent="Alice").compile(
    checkpointer=InMemorySaver())
```

A handoff tool returns a `Command(goto=agent, graph=Command.PARENT, update={...})`, which moves control to the other agent in the parent graph.

### 6. Hierarchical teams

Each team is a compiled supervisor graph, added as a node of the top-level supervisor:

```python
research_team = create_supervisor([searcher, scraper], model=llm).compile(name="research_team")
writing_team = create_supervisor([writer, editor], model=llm).compile(name="writing_team")
top = create_supervisor([research_team, writing_team], model=llm).compile()
```

### 7. Other frameworks

| Framework | Mental model | Coordination | Notes |
|---|---|---|---|
| **LangGraph** | Stateful graph (Pregel) | Edges, `Command`, `Send`, subgraphs | Low-level, durable, HITL, streaming, time travel |
| **CrewAI** | Crew of role-playing agents with tasks | `Process.sequential` / `Process.hierarchical` (manager LLM), Flows | Fast to prototype, opinionated |
| **AutoGen / AG2** | Agents conversing | `RoundRobinGroupChat`, `SelectorGroupChat`, `Swarm` (AgentChat) | Conversation-centric, good for research |
| **OpenAI Agents SDK** | Agents + handoffs + guardrails | `handoffs=[...]`, agents-as-tools, `Runner.run` | Lightweight, built-in tracing |
| **Google ADK** | Agent hierarchy | `SequentialAgent`, `ParallelAgent`, `LoopAgent`, LLM transfer | Gemini/Vertex integration |
| **Semantic Kernel** | Plugins + agents | Agent group chat / orchestrations | .NET/enterprise |

CrewAI sketch:

```python
from crewai import Agent, Task, Crew, Process
researcher = Agent(role="Researcher", goal="Find facts", backstory="...", tools=[search])
writer = Agent(role="Writer", goal="Write report", backstory="...")
t1 = Task(description="Research {topic}", expected_output="bullet facts", agent=researcher)
t2 = Task(description="Write a report", expected_output="markdown", agent=writer, context=[t1])
Crew(agents=[researcher, writer], tasks=[t1, t2], process=Process.sequential).kickoff(
    inputs={"topic": "LLM agents"})
```

### 8. Design checklist

1. **Contracts**: define each agent's input and output schema (Pydantic). Hand off structured results, not raw transcripts.
2. **State scoping**: decide which keys are shared and which are private (subgraphs). Use reducers for shared lists.
3. **Termination**: max iterations per agent, `recursion_limit`, and explicit FINISH criteria.
4. **Guardrails**: per-agent tool permissions, and approval before side effects.
5. **Observability**: LangSmith/OpenTelemetry traces, per-agent tags, token cost per agent.
6. **Evaluation**: test agents individually (unit evals), then end-to-end trajectories (did the router pick the right agent?).
7. **Failure handling**: retries (`RetryPolicy`), fallbacks to a simpler model, escalation to humans.
8. **Latency**: parallelise with `Send`, stream intermediate updates, and use small models for routing.

## Common Pitfalls

- **Agent sprawl.** Ten agents where two would do, which multiplies cost and failure points.
- **Context loss on handoff.** The next agent lacks the facts it needs. Pass structured summaries.
- **Context bloat.** Sharing every agent's full tool-call history pollutes the others. Use `output_mode="last_message"` or private subgraph state.
- **Ping-pong loops** between agents with no exit condition.
- **Using the LLM for routing where an `if` would do.**
- **No per-agent evaluation**, which makes it impossible to tell which agent caused a bad answer.

## Likely Follow-up Questions

1. **Supervisor vs swarm?** A supervisor gives central control and policy enforcement at the cost of an extra hop per turn. A swarm hands off directly, which has lower latency and fits conversational flows, but global rules are harder to enforce.
2. **How do agents share memory?** Shared graph state for the current run, a Store for cross-session facts, and private subgraph state for scratch work.
3. **How do you prevent infinite loops?** Iteration counters in state, `recursion_limit` in config, and termination criteria in the router.
4. **How do you scale to many agents?** Hierarchies of teams, plus dynamic tool and agent retrieval instead of listing all of them in the prompt.
5. **LangGraph vs CrewAI?** LangGraph is lower-level: explicit control, persistence, HITL, production-grade. CrewAI is a faster high-level abstraction with less control.

[← Back to questions](../questions.md)
