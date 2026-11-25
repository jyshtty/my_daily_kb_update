# LLM Observability

## What It Is
Seeing **what your LLM app did and why**: every prompt, model call, retrieval, tool call, token count, latency and cost, linked into a trace per request.

```
Trace: "user asks for refund"
 ├─ span: retrieve_docs        120 ms   5 chunks
 ├─ span: llm.chat (gpt-4o)    1.8 s    in 2,100 tok / out 230 tok   $0.008
 ├─ span: tool.lookup_order    90 ms    ok
 └─ span: llm.chat             900 ms   final answer
```

## Why Traditional APM Isn't Enough
| APM tells you | LLM apps also need |
|---|---|
| Latency, errors, throughput | **Prompt and response content** |
| HTTP status | **Quality** (was the answer right / grounded?) |
| CPU/memory | **Tokens and cost** per request/user/feature |
| Stack traces | **Agent trajectory** (which tools, how many steps) |

## What to Capture

| Category | Fields |
|---|---|
| Request | Trace ID, user/session ID, feature, prompt version |
| Model call | Model + version, parameters (temperature, max tokens), input/output messages |
| Usage | Input/output/cached tokens, cost |
| Latency | Total, **TTFT** (time to first token), tokens/sec |
| RAG | Query, retrieved doc IDs + scores, reranker scores |
| Tools | Name, arguments, result, errors, duration |
| Outcome | User feedback (👍/👎), eval scores, guardrail interventions |

## Key Metrics and Alerts

| Metric | Why |
|---|---|
| p50/p95/p99 latency, TTFT | User experience |
| Error rate, rate-limit (429) rate | Reliability |
| Tokens & **cost per request / user / day** | Budget, runaway agents |
| Steps per agent run | Loop detection |
| Guardrail block rate | Attacks or over-blocking |
| Online eval scores (faithfulness, relevance) | Quality drift |
| Thumbs-down rate | User-perceived quality |
| Cache hit rate (prompt cache, semantic cache) | Cost efficiency |

## Standards and Tools

| Tool | Notes |
|---|---|
| **OpenTelemetry GenAI semantic conventions** | Vendor-neutral attribute names for LLM spans (`gen_ai.*`) |
| **OpenLLMetry / OpenInference** | OTel-based auto-instrumentation for LLM SDKs and frameworks |
| **LangSmith** | Tracing, datasets, evals; native for LangChain/LangGraph |
| **Langfuse** | Open-source tracing, prompt management, evals, cost tracking |
| **Arize Phoenix** | Open-source tracing + evals (OpenInference) |
| **Datadog / New Relic LLM observability** | LLM monitoring inside existing APM |
| **Helicone** | Proxy-based logging, caching, cost |

## Example: Tracing with Langfuse Decorator
```python
from langfuse import observe

@observe()
def retrieve(query): ...

@observe()
def answer(question: str):
    docs = retrieve(question)
    return llm.invoke(build_prompt(question, docs))   # LLM call captured via integration
```

## From Traces to Improvement
```
Production traces → sample + score (LLM judge / feedback)
      → find failure clusters (error analysis)
      → add cases to eval dataset → fix → re-run evals → deploy
```
Observability and evals form one loop.

## Privacy and Cost of Logging
- **PII**: redact or mask before storing prompts; restrict access; set retention limits
- Regulated data (HIPAA, GDPR): self-hosted tools or region-pinned storage
- Sample full payloads at high volume; always keep metrics

## Pitfalls
- Logging only errors — most LLM failures return HTTP 200 with a bad answer
- No prompt version on traces → can't tie regressions to changes
- Storing raw PII in third-party tools
- No per-feature cost attribution → surprise bills

## Interview Quick Answer
> I trace every request end to end — prompts, model and version, retrievals, tool calls, tokens, cost, TTFT and latency — using OpenTelemetry-compatible tools such as LangSmith, Langfuse or Phoenix. I monitor latency, error and 429 rates, cost per user and feature, agent step counts, guardrail block rates and online eval scores, redact PII before logging, and feed sampled failing traces into the eval dataset.

## Rule of Thumb
If you can't see the prompt, the context and the cost for a request, you can't debug or improve it.

## Related
- [Q23 Agent observability](../hard_questions/ans/23_agent_observability.md)
- [Q5 Diagnosing high latency](../hard_questions/ans/05_diagnosing_high_latency.md)
