# Structured Output Strategies

Getting structured data from an LLM is a core production challenge. There are four main approaches.

## 1. Manual parsing (StrOutputParser + regex/json.loads)

```python
chain = prompt | llm | StrOutputParser()
raw = chain.invoke(input)
data = json.loads(raw)  # fragile — will throw on malformed output
```

Best for: quick prototypes. Bad for: production.

## 2. JSON prompt + JsonOutputParser

```python
from langchain_core.output_parsers import JsonOutputParser

chain = prompt | llm | JsonOutputParser()
```

Tells the LLM to return JSON; parser calls `json.loads`. No schema validation — you get a raw `dict`.

## 3. PydanticOutputParser (schema-driven)

```python
from langchain.output_parsers import PydanticOutputParser
from langchain_core.pydantic_v1 import BaseModel, Field

class Result(BaseModel):
    answer: str = Field(description="The answer")
    confidence: float = Field(description="0.0 to 1.0")

parser = PydanticOutputParser(pydantic_object=Result)
chain = prompt | llm | parser
result = chain.invoke(input)
print(result.confidence)  # typed float
```

The parser injects a JSON schema into the prompt via `get_format_instructions()`.

## 4. `.with_structured_output()` (modern approach)

```python
structured_llm = llm.with_structured_output(Result)
result = structured_llm.invoke("Give me an answer with confidence.")
```

Uses the model's native tool-calling/function-calling feature instead of prompt engineering. More reliable than PydanticOutputParser for modern models.

## Comparison

| Approach                    | Type safety | Reliability | Complexity |
|----------------------------|-------------|-------------|------------|
| Manual json.loads           | None        | Low         | Low        |
| JsonOutputParser            | None (dict) | Medium      | Low        |
| PydanticOutputParser        | High        | Medium      | Medium     |
| `.with_structured_output()` | High        | High        | Low        |

## Handling failures

```python
from langchain.output_parsers import OutputFixingParser

safe_parser = OutputFixingParser.from_llm(parser=parser, llm=llm)
```

On parse failure, sends the bad output + error back to the LLM and asks it to fix the format.

## Common interview questions

**Q: What is the most reliable way to get structured output from an LLM?**  
`.with_structured_output()` — it uses the model's native function-calling API, bypassing fragile JSON-in-text parsing.

**Q: Why is XML sometimes better than JSON for LLM output?**  
JSON breaks when field values contain quotes or newlines. XML uses tags as delimiters, making it more robust for long text content.

**Q: What does `get_format_instructions()` do?**  
Generates a human-readable JSON schema description to inject into the prompt, telling the LLM exactly what structure to return.
