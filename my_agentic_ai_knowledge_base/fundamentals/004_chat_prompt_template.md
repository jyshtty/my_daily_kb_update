# ChatPromptTemplate

`ChatPromptTemplate` builds a reusable prompt with named parameters that gets injected at runtime.

## Usage

```python
import textwrap
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("Tell me a short story about {subject}")

chain = prompt | llm_openAI | StrOutputParser()

story = chain.invoke({"subject": "cat"})
print(textwrap.fill(story, 80))
```

## How it works

1. `from_template` creates a prompt with `{subject}` as a placeholder.
2. When the chain is invoked with a dict, the placeholder is replaced with the provided value.
3. The formatted prompt is passed to the LLM, and `StrOutputParser` returns the plain string response.

## When to use

- When the same prompt structure is reused with different inputs.
- As the first step in an LCEL chain before the LLM and output parser.
