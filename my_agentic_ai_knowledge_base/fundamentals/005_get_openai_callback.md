# get_openai_callback

`get_openai_callback` is a context manager that tracks token usage and cost for OpenAI LLM calls.

## Usage

```python
from langchain_community.callbacks import get_openai_callback

with get_openai_callback() as cb:
    result = chain.invoke({"subject": "dog"})

print(f"Tokens used: {cb.total_tokens}")
print(f"Prompt tokens: {cb.prompt_tokens}")
print(f"Completion tokens: {cb.completion_tokens}")
print(f"Cost (USD): {cb.total_cost}")
```

## Available attributes on `cb`

| Attribute            | Description                        |
|---------------------|------------------------------------|
| `total_tokens`      | Total tokens consumed               |
| `prompt_tokens`     | Tokens in the prompt                |
| `completion_tokens` | Tokens in the response              |
| `total_cost`        | Estimated cost in USD               |

## When to use

- Debugging token consumption during development.
- Estimating costs before deploying a chain to production.
