# PromptTemplate vs ChatPromptTemplate

LangChain has two main prompt template classes. The choice depends on the model type.

## PromptTemplate

For plain-text (completion) models. Produces a single string.

```python
from langchain_core.prompts import PromptTemplate

prompt = PromptTemplate(
    template="Tell me about {topic}.",
    input_variables=["topic"],
)
prompt.format(topic="LLMs")
# → "Tell me about LLMs."
```

Use `partial_variables` to pre-fill some variables at creation time:

```python
prompt = PromptTemplate(
    template="Answer in {language}: {question}",
    input_variables=["question"],
    partial_variables={"language": "English"},
)
```

## ChatPromptTemplate

For chat models (OpenAI, Anthropic, etc.). Produces a list of messages (system / human / ai).

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("human", "{question}"),
])
prompt.invoke({"question": "What is LCEL?"})
```

### `from_template` shorthand

```python
prompt = ChatPromptTemplate.from_template("Tell me a story about {subject}")
```
Creates a single `HumanMessage` template.

## Key differences

| Feature                  | PromptTemplate         | ChatPromptTemplate          |
|--------------------------|------------------------|-----------------------------|
| Output type              | Single string          | List of `BaseMessage`       |
| Suitable for             | Completion models      | Chat models                 |
| Multi-turn support       | No                     | Yes (system + human + ai)   |

## Common interview questions

**Q: When would you use `PromptTemplate` over `ChatPromptTemplate`?**  
When using a completion (non-chat) model, or when the downstream component expects a plain string rather than a message list.

**Q: What is `partial_variables`?**  
Variables resolved at template creation time, not at invoke time. Useful for injecting static values like format instructions or language settings once.
