# FastMCP vs LangChain @tool

## Comparison

| | FastMCP | @tool (LangChain) |
|---|---|---|
| **Purpose** | Expose tools as an MCP server | Use tools within LangChain agents/chains |
| **Interoperability** | Any MCP-compatible client (Claude, Cursor, etc.) | LangChain ecosystem only |
| **Complexity** | Slightly more setup (server) | Simple decorator |
| **Best for** | Sharing tools across multiple AI apps | Building LangChain-specific pipelines |

## Rule of Thumb

- Use `FastMCP` if you want tools accessible **outside LangChain** (e.g., Claude Desktop, Cursor)
- Use `@tool` if you are staying **within LangChain** agents/chains

## Code Examples

### FastMCP
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-server")

@mcp.tool()
def search_docs(query: str) -> str:
    """Search documentation"""
    ...

mcp.run()
```

### LangChain @tool
```python
from langchain_core.tools import tool

@tool
def search_docs(query: str) -> str:
    """Search documentation"""
    ...

# Used inside a LangChain agent
tools = [search_docs]
agent = create_react_agent(llm, tools, prompt)
```

## When to Use Each

Use `FastMCP` when:
- You want tools accessible from Claude Desktop, Cursor, or any MCP client
- You are building a tool server shared across multiple AI applications

Use `@tool` when:
- You are building a LangChain ReAct agent with a tool-calling loop
- Tools are only needed inside LangChain pipelines

