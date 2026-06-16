# LCEL — LangChain Expression Language

LCEL is LangChain's declarative way to compose chains using the `|` (pipe) operator.

## How it works

```python
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"subject": "cat"})
```

Each component implements `Runnable`. The `|` operator connects them so output of the left flows as input to the right.

## Key Runnable methods

| Method         | Description                                      |
|---------------|--------------------------------------------------|
| `invoke`      | Run chain with a single input, return output     |
| `batch`       | Run with a list of inputs, return list of outputs |
| `stream`      | Stream output tokens as they arrive              |
| `ainvoke`     | Async version of invoke                          |

## Why LCEL over plain function calls

- Built-in streaming, batching, and async support.
- Automatic retries and fallbacks with `.with_retry()` / `.with_fallbacks()`.
- Easy observability via LangSmith tracing.
- Composable — chains can be nested inside other chains.

## Common interview questions

**Q: What does the `|` operator do in LangChain?**  
Connects two `Runnable` objects. Output of the left becomes input of the right.

**Q: What is the difference between `invoke`, `batch`, and `stream`?**  
`invoke` → single input/output. `batch` → list of inputs in parallel. `stream` → yields tokens incrementally.

**Q: Can you add logic/branching inside an LCEL chain?**  
Yes, using `RunnableLambda` (wrap any Python function) or `RunnableBranch` (conditional routing).
