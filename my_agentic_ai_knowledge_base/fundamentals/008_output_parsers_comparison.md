# Output Parsers — Comparison

LangChain provides several output parsers. Choose based on how structured your required output is.

## Overview

| Parser                  | Output type        | Use when                                      |
|------------------------|--------------------|-----------------------------------------------|
| `StrOutputParser`      | `str`              | You just need the raw text                    |
| `JsonOutputParser`     | `dict`             | You need JSON but no schema enforcement       |
| `PydanticOutputParser` | Pydantic model     | You need typed, validated structured output   |
| `XMLOutputParser`      | `dict` (from XML)  | Output contains long text that breaks JSON    |
| `CommaSeparatedListOutputParser` | `list[str]` | Simple list extraction                   |
| `OutputFixingParser`   | Wraps any parser   | Auto-retry with error feedback on parse fail  |

## When to prefer XML over JSON

JSON is fragile with long text — quotes and newlines inside field values corrupt the structure. XML tolerates long text better because it uses tags instead of quotes as delimiters.

## Handling parser failures

```python
from langchain.output_parsers import OutputFixingParser

robust_parser = OutputFixingParser.from_llm(parser=parser, llm=llm)
chain = prompt | llm | robust_parser
```

`OutputFixingParser` catches parse errors and sends the bad output + error message back to the LLM asking it to fix the format.

## Common interview questions

**Q: Which output parser would you use for a production pipeline that needs typed data?**  
`PydanticOutputParser` — it enforces a schema and returns a typed object, making downstream code safer.

**Q: Why might JSON output from an LLM fail to parse?**  
The LLM may include quotes inside string values, trailing commas, or extra text outside the JSON block — all of which break `json.loads`.

**Q: What is `RetryWithErrorOutputParser`?**  
Similar to `OutputFixingParser` but also passes the original prompt back to the LLM along with the error, giving more context for correction.
