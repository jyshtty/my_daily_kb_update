# Guardrails: Fundamentals

## What Guardrails Are
**Guardrails** are controls around an LLM that keep inputs, outputs and actions **safe, compliant and on-task**. The model itself is not a security boundary — guardrails are the layers you add around it.

```
User input → [Input guardrails] → LLM (+ tools, RAG) → [Output guardrails] → User
                                       │
                               [Action / tool guardrails]
```

## Layers

| Layer | Checks | Examples |
|---|---|---|
| **Input** | What goes into the model | Prompt-injection/jailbreak detection, topic restriction, PII redaction, length limits, language check |
| **Retrieval** | What the model reads | Access control on documents, scanning retrieved text for injected instructions |
| **Output** | What comes back | Toxicity, PII leakage, hallucination / grounding checks, schema validation, competitor/off-brand content |
| **Action (tool)** | What the agent does | Allow-lists, argument validation, least privilege, rate limits, **human approval** for risky actions |
| **System** | Around everything | AuthN/AuthZ, audit logging, monitoring, kill switch |

## Types of Guardrails

| Type | Implementation | Speed / cost |
|---|---|---|
| **Rule-based** | Regex, keyword lists, allow/deny lists, JSON schema | Very fast, cheap, brittle |
| **Classifier-based** | Small ML models: toxicity, PII NER, injection classifiers (e.g. Llama Guard, Prompt Guard, Presidio) | Fast, good accuracy |
| **LLM-based** | An LLM checks policy compliance or groundedness | Flexible, slower, costs tokens |
| **Structural** | Constrained decoding, function-calling schemas, Pydantic validation | Guarantees format |
| **Process** | Human-in-the-loop, approvals, escalation | Slowest, strongest for high stakes |

Combine them: cheap rules first, classifiers next, LLM checks only where needed.

## Common Guardrail Policies
- **Content safety**: hate, harassment, sexual, violence, self-harm
- **Prompt attacks**: jailbreaks, direct and indirect injection (see `006_prompt_injection_and_jailbreaks.md`)
- **Sensitive data**: block or mask PII/PHI, secrets, API keys
- **Topic control**: stay in scope (e.g. no medical/legal/investment advice)
- **Grounding**: answer only from retrieved sources; refuse otherwise
- **Format**: valid JSON / schema, length limits
- **Brand & compliance**: no competitor mentions, required disclaimers

## Example: Layered Pipeline
```python
def answer(user_id: str, text: str) -> str:
    if len(text) > 4000:
        return "Message too long."
    if injection_classifier(text) > 0.9:
        log_security_event(user_id, text)
        return "I can't help with that request."
    text = redact_pii(text)                          # Presidio / regex

    docs = retrieve(text, allowed_for=user_id)       # access control at retrieval
    reply = llm.generate(system=POLICY_PROMPT, user=text, context=docs)

    if moderation(reply).flagged or leaks_pii(reply):
        return SAFE_FALLBACK
    if grounding_score(reply, docs) < 0.7:
        return "I couldn't find a reliable answer in our documents."
    return reply
```

## Blocking vs Other Actions
| Action | When |
|---|---|
| **Block** with a safe message | Clear violation |
| **Mask / redact** | PII in otherwise valid content |
| **Rewrite / regenerate** | Format or tone issue |
| **Escalate to human** | High-stakes or uncertain cases |
| **Log only (shadow mode)** | Tuning a new guardrail before enforcing |

## Measuring Guardrails
Guardrails are classifiers — evaluate them:
- **False negative rate** (harmful content let through) — safety risk
- **False positive rate** (good requests blocked) — user frustration
- Added **latency** and **cost**
- Test with red-team datasets and normal traffic; roll out in shadow mode first

## Latency Tips
- Run input checks **in parallel** with the main LLM call; cancel if the check fails
- Stream output but check in chunks; hold back final release until checks pass for high-risk apps
- Use small classifiers over LLM judges on the hot path

## Pitfalls

| Pitfall | Fix |
|---|---|
| Relying on the system prompt alone | Enforce in code outside the model |
| Only input checks | Check outputs and actions too |
| Over-blocking | Measure false positives, tune thresholds, shadow mode |
| Agents with broad permissions | Least privilege + approval for destructive actions |
| No monitoring | Log violations, alert on spikes, review regularly |

## Interview Quick Answer
> Guardrails are layered controls around the model: input checks for injection, PII and topic scope; access control on retrieval; output checks for toxicity, PII leakage, grounding and schema; and action controls like least privilege, argument validation and human approval for risky tool calls. I combine fast rules and classifiers with LLM checks where needed, evaluate guardrails for false positives and negatives, roll them out in shadow mode, and monitor violations in production.

## Rule of Thumb
Never trust the model to police itself: enforce policy in code, at every layer, and measure both what you block and what slips through.

## Related
- [Q16 Guardrails](../hard_questions/ans/16_guardrails.md)
- [Q8 Bedrock guardrails](../easy_questions/ans/08_bedrock_guardrails.md)
- [Q4 Agent approval for sensitive actions](../easy_questions/ans/04_agent_approval_sensitive_actions.md)
