# Guardrail Tools and Frameworks

## Landscape

| Tool | Type | What it does |
|---|---|---|
| **NVIDIA NeMo Guardrails** | Open-source framework | Programmable "rails" (input, output, dialog, retrieval, execution) defined in config + **Colang**; topic control, jailbreak and fact-checking rails |
| **Guardrails AI** | Open-source Python library | Wraps LLM calls with **validators** (from Guardrails Hub): PII, toxicity, regex, schema, competitor check; can re-ask or fix on failure |
| **Llama Guard** (Meta) | Open safety classifier model | Classifies prompts and responses against a hazard taxonomy (safe / unsafe + category) |
| **Prompt Guard** (Meta) | Small classifier | Detects prompt injection and jailbreak attempts |
| **Amazon Bedrock Guardrails** | Managed service | Content filters, denied topics, word filters, PII filters, contextual grounding; `ApplyGuardrail` API works with any model |
| **Azure AI Content Safety** | Managed service | Harm categories, **Prompt Shields** (direct + indirect injection), groundedness detection, protected material |
| **OpenAI Moderation API** | Managed endpoint | Free classification of text/images into harm categories |
| **Google Model Armor / Vertex safety filters** | Managed | Prompt/response screening, safety settings |
| **Microsoft Presidio** | Open-source | PII detection and anonymization (NER + regex) |
| **LLM Guard** (Protect AI) | Open-source | Input/output scanners: injection, PII, secrets, toxicity, ban topics |
| **Pydantic / Instructor / structured outputs** | Libraries / API features | Enforce JSON schema, validate and retry |

## NeMo Guardrails Example
```yaml
# config.yml
models:
  - type: main
    engine: openai
    model: gpt-4o-mini
rails:
  input:
    flows:
      - self check input        # LLM checks input against policy
  output:
    flows:
      - self check output
```
```colang
define user ask about politics
  "What do you think about the election?"
  "Who should I vote for?"

define flow politics
  user ask about politics
  bot refuse politics

define bot refuse politics
  "I can only help with questions about our products."
```
```python
from nemoguardrails import LLMRails, RailsConfig
rails = LLMRails(RailsConfig.from_path("./config"))
reply = rails.generate(messages=[{"role": "user", "content": "Who should I vote for?"}])
```

## Guardrails AI Example
```python
from guardrails import Guard
from guardrails.hub import DetectPII, ToxicLanguage   # installed from Guardrails Hub

guard = Guard().use_many(
    DetectPII(pii_entities=["EMAIL_ADDRESS", "PHONE_NUMBER"], on_fail="fix"),
    ToxicLanguage(threshold=0.5, on_fail="exception"),
)
result = guard.validate(llm_output)      # fixes PII, raises on toxicity
```

## Llama Guard Usage Pattern
```
prompt  → Llama Guard → "safe" | "unsafe\nS1" (category code) → block or continue
response→ Llama Guard → same check on output
```
Self-hostable, so data stays in your environment; customize categories in the prompt.

## Bedrock ApplyGuardrail (model-independent)
```python
resp = bedrock_runtime.apply_guardrail(
    guardrailIdentifier="gr-abc123", guardrailVersion="1",
    source="INPUT",
    content=[{"text": {"text": user_text}}],
)
if resp["action"] == "GUARDRAIL_INTERVENED":
    return resp["outputs"][0]["text"]     # configured blocked message
```

## Choosing

| Need | Good choice |
|---|---|
| Already on AWS / Azure, want managed | Bedrock Guardrails / Azure AI Content Safety |
| Self-hosted, data can't leave VPC | Llama Guard + Prompt Guard + Presidio, LLM Guard |
| Conversation flow and topic control | NeMo Guardrails |
| Validate and fix structured outputs | Guardrails AI, Pydantic/Instructor, native structured outputs |
| Quick free moderation | OpenAI Moderation API |
| Red-team testing | promptfoo, garak, PyRIT |

## Pitfalls
- One tool for everything — combine classifiers, rules and policy-in-code
- Adding several LLM-based rails in sequence → large latency; parallelize or use small classifiers
- Not evaluating false positives on real traffic
- Forgetting tool/action guardrails — these tools mostly cover text

## Interview Quick Answer
> I pick guardrail tools by deployment and need: managed services like Bedrock Guardrails or Azure Content Safety on those clouds; self-hosted Llama Guard, Prompt Guard, Presidio or LLM Guard when data can't leave the VPC; NeMo Guardrails for dialog and topic rails; and Guardrails AI or structured outputs for validating formats. I layer them with policy enforced in code for tool calls, and evaluate them for false positives and latency.

## Rule of Thumb
Use managed or classifier-based guardrails for content, schemas for format, and plain code for permissions.

## Related
- [005 Guardrails fundamentals](005_guardrails_fundamentals.md)
- [Q8 Bedrock guardrails](../easy_questions/ans/08_bedrock_guardrails.md)
