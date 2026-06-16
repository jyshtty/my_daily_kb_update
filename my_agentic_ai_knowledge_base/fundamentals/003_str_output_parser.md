# StrOutputParser

`StrOutputParser` converts an LLM response object into a plain Python string.

## Why use it

LLMs return message objects (e.g. `AIMessage`). Piping through `StrOutputParser` extracts just the text content, making the output easy to use downstream.

## Usage

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI

llm_openAI = ChatOpenAI()

chain = llm_openAI | StrOutputParser()

result = chain.invoke("Hello!")
print(result)  # plain string, e.g. "Hello! How can I help you today?"
```

## How it works

The `|` operator builds a LangChain **LCEL** (LangChain Expression Language) chain. When `invoke` is called:

1. The prompt string is sent to `llm_openAI`, which returns an `AIMessage`.
2. `StrOutputParser` calls `.content` on that message and returns the raw string.

## When to use

- Any time you want a string out of an LLM instead of a message object.
- As a building block before passing output to another chain step that expects a string.
