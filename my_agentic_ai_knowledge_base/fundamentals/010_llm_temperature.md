# LLM Temperature

Temperature controls the randomness of LLM output. It is one of the most common interview topics.

## What it does

At each token generation step, the model produces a probability distribution over possible next tokens. Temperature scales that distribution:

- **Low temperature (0–0.3)** → distribution sharpens → model picks the highest-probability token almost deterministically → consistent, factual answers.
- **High temperature (0.7–1.0+)** → distribution flattens → model samples more randomly → creative, varied, sometimes surprising answers.

## Practical values

| Value | Behaviour            | Use case                              |
|-------|----------------------|---------------------------------------|
| 0.0   | Deterministic        | Data extraction, classification, code |
| 0.3   | Mostly consistent    | Summarisation, Q&A                    |
| 0.7   | Balanced             | General chat                          |
| 1.0+  | Creative / random    | Brainstorming, creative writing       |

## LangChain default vs OpenAI default

> **Important interview trap:** LangChain's default temperature is **0.7**, not 0. OpenAI's API default is also 1.0.  
> Always set `temperature=0.0` explicitly for deterministic tasks.

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.0)  # deterministic
```

## Common interview questions

**Q: What is temperature in LLMs?**  
A parameter that controls output randomness. 0 = deterministic, higher = more creative/random.

**Q: What temperature would you use for a production data-extraction pipeline?**  
0.0 — you want consistent, reproducible results, not creative variation.

**Q: Does `temperature=0` guarantee identical outputs every time?**  
Not always — floating-point non-determinism in GPU operations can still introduce tiny variations, but output is nearly always identical.
