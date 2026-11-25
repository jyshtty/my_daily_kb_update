# LLM Evals: Fundamentals

## What Evals Are
**Evals** are systematic tests that measure whether an LLM application does what it should. They are the unit tests and benchmarks of AI engineering.

```
Dataset (inputs + expected behaviour)
      → Run app (prompt + model + retrieval + tools)
      → Score outputs (code checks / LLM judge / humans)
      → Compare with baseline → ship, fix, or roll back
```

Without evals, every prompt or model change is a guess.

## Why LLM Apps Need Them
| Traditional software | LLM apps |
|---|---|
| Deterministic output | Non-deterministic, many valid answers |
| Exact assertions (`==`) | Fuzzy correctness, tone, faithfulness |
| Bugs are reproducible | Behaviour shifts with prompt, model version, data |
| Unit tests are enough | Need datasets, metrics and judges |

## Types of Evals

| Type | When | Purpose |
|---|---|---|
| **Offline evals** | Before release, in CI | Compare versions on a fixed golden dataset |
| **Online evals** | In production | Score live traffic (sampled), catch drift |
| **Regression evals** | Every change | Make sure nothing that worked breaks |
| **Benchmark evals** | Model selection | Public benchmarks (MMLU, GPQA, SWE-bench) — only a rough signal |
| **Red-team / safety evals** | Before release, periodically | Jailbreaks, harmful content, PII leaks |
| **A/B tests** | In production | Real user outcomes between variants |

## Scoring Methods

| Method | Examples | Pros | Cons |
|---|---|---|---|
| **Code-based (deterministic)** | Exact match, regex, JSON schema valid, contains keyword, SQL executes, unit tests pass | Fast, cheap, reliable | Only for checkable properties |
| **Reference-based metrics** | BLEU, ROUGE, embedding similarity | Automatic | Weak correlation with quality for open-ended text |
| **LLM-as-a-judge** | Rubric scoring, pairwise comparison (see `004_llm_as_a_judge.md`) | Scales, handles nuance | Bias, cost, needs calibration |
| **Human evaluation** | SME review, thumbs up/down | Ground truth | Slow, expensive, inconsistent |

Prefer this order: **code checks first, LLM judge for the rest, humans to calibrate.**

## What to Measure

| Dimension | Example metric |
|---|---|
| Correctness | Accuracy vs reference answer |
| Faithfulness / groundedness | Claims supported by context |
| Relevance | Answers the actual question |
| Completeness | Covers all required points |
| Format | Valid JSON / schema compliance rate |
| Safety | Toxicity, PII leakage, policy violations |
| Tone / style | Rubric score |
| Task success | Agent achieved the goal |
| **Operational** | Latency (p50/p95, TTFT), tokens, cost per request, error rate |

## Building an Eval Dataset (Golden Set)
1. **Start from real data** — production logs, support tickets, user queries
2. Cover **happy paths, edge cases, adversarial inputs**, and known past failures
3. Add **reference answers** or grading criteria written by domain experts
4. Supplement with **synthetic** cases (LLM-generated), reviewed by humans
5. Tag cases by category / difficulty so you can slice results
6. Version the dataset; keep adding every production bug as a new case

Size guide: 20–50 cases to start iterating, 100–500+ for reliable comparisons.

## Eval-Driven Development Loop
```
     ┌──────── analyze failures (error analysis) ←──────┐
     ↓                                                    │
change prompt/model/retrieval → run evals → compare to baseline
     ↑                                                    │
     └──── add new failing cases from production ─────────┘
```
- Change **one thing at a time**
- Read the failures, not just the averages — **error analysis** finds the real problems
- Gate merges in CI on thresholds; block regressions

## Example: Pytest-Style Eval
```python
import json, pytest

CASES = json.load(open("evals/golden.json"))

@pytest.mark.parametrize("case", CASES, ids=lambda c: c["id"])
def test_extraction(case):
    out = app.run(case["input"])
    data = json.loads(out)                          # format check
    assert set(data) >= {"name", "amount", "date"}  # schema check
    assert data["amount"] == case["expected"]["amount"]  # correctness
```

## Tools

| Tool | Focus |
|---|---|
| **DeepEval** | Pytest-style metrics, G-Eval custom rubrics, CI |
| **Ragas** | RAG metrics (faithfulness, context precision/recall) |
| **promptfoo** | Config-driven prompt/model comparison, red teaming |
| **LangSmith** | Datasets, experiments, tracing, online evals |
| **Langfuse / Arize Phoenix** | Open-source tracing + evals |
| **OpenAI Evals, Inspect AI** | Eval frameworks / harnesses |
| **Braintrust** | Experiments, scoring, prompt playground |

## Pitfalls

| Pitfall | Fix |
|---|---|
| "Vibe checks" only | Build a dataset and metrics early |
| Only averages tracked | Slice by category, read individual failures |
| Dataset never updated | Feed production failures back in |
| Trusting an uncalibrated LLM judge | Measure agreement with human labels |
| Overfitting prompts to the eval set | Hold-out set, refresh cases |
| Ignoring cost and latency | Track them as first-class metrics |
| Non-determinism hides regressions | Run multiple samples, use temperature 0 for judges, compare distributions |

## Interview Quick Answer
> Evals are how I know an LLM app works. I build a golden dataset from real traffic plus edge and adversarial cases, score outputs with deterministic checks where possible and a calibrated LLM judge for subjective qualities, and track quality, safety, latency and cost. Evals run in CI to block regressions, and online evals sample production traffic. Every production failure becomes a new test case.

## Rule of Thumb
No eval, no ship. Start small with real examples, prefer deterministic checks, calibrate judges against humans, and keep growing the dataset from production.

## Related
- [Q7 Ragas / DeepEval](../easy_questions/ans/07_ragas_deepeval.md)
- [Q18 How do you know it works?](../hard_questions/ans/18_how_do_you_know_it_works.md)
- [Q22 Regression detection & rollback](../hard_questions/ans/22_regression_detection_rollback.md)
