# LangChain Message Types

Chat models communicate via a sequence of typed messages. Understanding the message types is fundamental.

## Types

| Class            | Role      | Description                                          |
|-----------------|-----------|------------------------------------------------------|
| `SystemMessage` | `system`  | Sets the model's persona, rules, and context         |
| `HumanMessage`  | `user`    | Input from the user                                  |
| `AIMessage`     | `assistant` | Response from the model                            |
| `FunctionMessage` | `function` | Result of a tool/function call (legacy)           |
| `ToolMessage`   | `tool`    | Result of a tool call (modern, replaces FunctionMessage) |

## Usage

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage(content="You are a helpful assistant."),
    HumanMessage(content="What is LCEL?"),
]

response = llm.invoke(messages)
# response is an AIMessage
print(response.content)       # the text
print(response.response_metadata)  # token usage, model name, finish reason
```

## AIMessage attributes

```python
response.content              # str — the reply text
response.response_metadata    # dict — token usage, finish_reason, model_name
response.id                   # run ID
response.tool_calls           # list — populated when the model calls a tool
```

## Common interview questions

**Q: What is the difference between `SystemMessage` and `HumanMessage`?**  
`SystemMessage` sets the model's behaviour and context before the conversation starts. `HumanMessage` represents what the user says.

**Q: What does `llm.invoke("Hello")` return?**  
An `AIMessage` object. Access `.content` for the plain text string, or pipe through `StrOutputParser()` to get a string directly.

**Q: When would you use `ToolMessage`?**  
When a model calls a tool (function), you execute it and return the result as a `ToolMessage` so the model can continue reasoning with that result.
