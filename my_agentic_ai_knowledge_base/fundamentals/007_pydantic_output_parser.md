# PydanticOutputParser

`PydanticOutputParser` instructs the LLM to return JSON matching a Pydantic model and automatically parses the response into that model instance.

## Usage

```python
from langchain_core.pydantic_v1 import BaseModel, Field
from langchain.output_parsers import PydanticOutputParser
from langchain_core.prompts import PromptTemplate

class PairItem(BaseModel):
    first_item:  str   = Field(description="Item from the first list")
    second_item: str   = Field(description="Relevant item if exists")
    score:       float = Field(description="Score of relevance")
    explanation: str   = Field(description="Explain your decision")

class PairedList(BaseModel):
    pair_list: list[PairItem]

parser = PydanticOutputParser(pydantic_object=PairedList)

prompt = PromptTemplate(
    template="Match the lists.\n{first_list}\n{second_list}\n{format_instructions}",
    input_variables=["first_list", "second_list"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)

chain = prompt | llm | parser

result = chain.invoke({"first_list": "...", "second_list": "..."})
# result is a PairedList instance — access fields directly
for item in result.pair_list:
    print(item.first_item, item.score)
```

## How it works

1. `parser.get_format_instructions()` generates a JSON schema description injected into the prompt.
2. The LLM returns JSON conforming to that schema.
3. The parser deserializes the JSON into the Pydantic model.

## Important caveats

- Does **not** guarantee valid output — LLM hallucinations can still produce malformed JSON, causing a `ValidationError` or `OutputParserException`.
- Use `OutputFixingParser` or `RetryWithErrorOutputParser` to auto-correct failures.

## Common interview questions

**Q: How does PydanticOutputParser differ from StrOutputParser?**  
`StrOutputParser` returns a plain string. `PydanticOutputParser` validates and deserializes the response into a typed Python object.

**Q: What is `partial_variables` in PromptTemplate?**  
Pre-filled template variables resolved at prompt creation time, not at invoke time. Used here to inject format instructions once rather than on every call.

**Q: What happens if the LLM returns invalid JSON?**  
A `langchain.schema.OutputParserException` is raised. Wrap with `OutputFixingParser` to send the error back to the LLM for self-correction.
