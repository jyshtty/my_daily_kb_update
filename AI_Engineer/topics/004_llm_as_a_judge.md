# LLM-as-a-Judge

## What It Is
Using an LLM to **grade** another LLM's output against a rubric, a reference answer, or another output.

```
(input, output, [reference/context], rubric) → Judge LLM → score + reasoning
```
It scales subjective evaluation (helpfulness, tone, faithfulness) that code can't check.

## Judge Formats

| Format | How | Best for |
|---|---|---|
| **Pointwise (direct scoring)** | Score one output 1–5 or pass/fail | Absolute quality gates, monitoring |
| **Pairwise comparison** | "Which is better, A or B?" | Comparing prompts/models; more reliable than absolute scores |
| **Reference-based** | Compare output with a gold answer | Correctness |
| **Reference-free** | Check against context / rubric only | Faithfulness, policy compliance, production traffic |

## Writing a Good Judge Prompt
- **One criterion per judge** (faithfulness, tone, completeness) — not "rate overall quality"
- Prefer **binary or small scales** (pass/fail, 1–3) with clear definitions for each level
- Give **examples** of each grade (few-shot)
- Ask for **reasoning before the verdict** (chain-of-thought improves reliability)
- Return **structured output** (JSON) for parsing

```text
You are grading whether an answer is fully supported by the context.

Context: {context}
Answer: {answer}

Steps:
1. List each factual claim in the answer.
2. For each claim, say whether the context supports it.
3. Verdict: PASS only if every claim is supported.

Return JSON: {"reasoning": "...", "verdict": "PASS" | "FAIL"}
```

## Known Biases

| Bias | Description | Mitigation |
|---|---|---|
| **Position bias** | Prefers the first (or second) option | Swap order, run both, count only consistent wins |
| **Verbosity bias** | Prefers longer answers | Rubric says length is not quality; control for length |
| **Self-preference** | Favors outputs from its own model family | Use a different model family as judge |
| **Leniency / score clustering** | Gives 4/5 to everything | Binary verdicts, concrete criteria, examples |
| **Sycophancy to confident tone** | Rewards confident but wrong answers | Reference answers, claim-by-claim checks |

## Calibrating the Judge
Treat the judge like a model you must validate:
1. Humans (SMEs) label 50–200 samples
2. Run the judge on the same samples
3. Measure agreement: accuracy, precision/recall on FAIL, **Cohen's kappa**
4. Iterate on the judge prompt until agreement is acceptable (e.g. κ > 0.6–0.7)
5. **Pin** the judge model version and prompt; re-calibrate when you change either

## Cost and Speed
- Use a strong model for judging critical criteria; smaller/cheaper judges for high-volume monitoring after calibration
- Sample production traffic instead of judging everything
- Cache judgments for unchanged (input, output) pairs
- Specialised judge/evaluator models exist (e.g. Prometheus, fine-tuned classifiers)

## G-Eval (DeepEval)
Define criteria in plain language; the framework generates evaluation steps and a normalized score.
```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams

policy = GEval(
    name="Policy compliance",
    criteria="The response must not promise refunds beyond 30 days or give legal advice.",
    evaluation_params=[LLMTestCaseParams.INPUT, LLMTestCaseParams.ACTUAL_OUTPUT],
    threshold=0.7,
)
```

## When Not to Use a Judge
- The property is checkable by code (valid JSON, exact number, test passes) → use code
- Extremely high-stakes decisions → humans in the loop
- Judge can't know the facts (no context or reference) → it will guess

## Interview Quick Answer
> LLM-as-a-judge scores outputs with a rubric, either pointwise or pairwise. I keep one criterion per judge, use binary or small scales with examples, ask for reasoning before the verdict, and return JSON. Judges have position, verbosity and self-preference biases, so I swap orders, use a different model family, and calibrate against human labels with Cohen's kappa before trusting them. I pin the judge version and re-calibrate on changes.

## Rule of Thumb
A judge is a model too: validate it against humans before you let it gate releases.

## Related
- [Q19 LLM as a judge](../hard_questions/ans/19_llm_as_a_judge.md)
