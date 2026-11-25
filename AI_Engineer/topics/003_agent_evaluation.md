# Agent Evaluation

## Why Agents Are Harder to Evaluate
An agent takes **many steps**: it plans, calls tools, reads results and decides again. The final answer can be right for the wrong reasons, or wrong because of one bad step.

```
Goal → [plan] → tool A → [reason] → tool B → [reason] → final answer / action
          ↑ evaluate each step (trajectory) + the final outcome
```

## Three Levels

| Level | What you check | Examples |
|---|---|---|
| **Final outcome** | Did it achieve the goal? | Task success rate, correct DB state, ticket resolved |
| **Trajectory** | Was the path sensible? | Right tools, right order, no loops, no unnecessary calls |
| **Single step** | Was each decision right? | Correct tool chosen, valid arguments, correct parsing of the result |

## Key Metrics

| Metric | Description |
|---|---|
| **Task success rate** | % of tasks completed correctly (the headline metric) |
| **Tool selection accuracy** | Chose the expected tool for a step |
| **Tool argument correctness** | Arguments valid and match the intent (schema + values) |
| **Trajectory match** | Exact, in-order, any-order or subset match against a reference path |
| **Steps / tool calls per task** | Efficiency; flags loops |
| **Recovery rate** | Recovered after a tool error |
| **Safety violations** | Attempted forbidden or unapproved actions |
| **Escalation correctness** | Handed off to a human when it should |
| **Cost and latency per task** | Tokens, $, wall-clock |
| **pass^k (consistency)** | Succeeds on all of k repeated runs — reliability, not luck |

## Outcome-Based Checks (Prefer These)
Check the **end state** of the environment rather than the text:
```python
def test_refund_agent():
    env = make_sandbox(order_id=42, status="delivered")
    agent.run("Refund order 42, it arrived broken", env=env)
    assert env.db.orders[42].status == "refunded"     # state check
    assert env.payments.refunds == [42]               # exactly once
    assert not env.emails.sent_to_wrong_user()
```
Use sandboxes or mocked tools so tests are repeatable and safe.

## Trajectory Evaluation
```python
expected = ["lookup_order", "check_refund_policy", "issue_refund"]
actual   = [step.tool for step in run.tool_calls]

exact_match    = actual == expected
in_order_match = is_subsequence(expected, actual)      # extra steps allowed
any_order      = set(expected) <= set(actual)
```
- Several valid paths often exist → prefer subset / in-order match, or an LLM judge with a rubric for "was the trajectory reasonable?"
- LangSmith and the `agentevals` package provide trajectory evaluators (strict, unordered, subset, LLM-judge)

## Simulated Users
For conversational agents, an LLM plays the user with a goal and persona; the run ends when the goal is met or a turn limit is hit. Then score outcome + policy compliance (as in τ-bench style evaluations).

## Benchmarks (for model choice, not your app)
| Benchmark | Measures |
|---|---|
| SWE-bench (Verified) | Fixing real GitHub issues |
| τ-bench / τ²-bench | Tool use + policy following with simulated users |
| GAIA | General assistant tasks with tools |
| WebArena / OSWorld | Web / computer-use tasks |
| BFCL | Function-calling accuracy |

Your own task-specific suite matters more than public benchmarks.

## Online Monitoring for Agents
- Trace every run (LLM calls, tool calls, inputs/outputs) — see `008_llm_observability.md`
- Alert on step-count spikes, tool error rate, loop detection, cost per run
- Sample traces for LLM-judge or human review; turn failures into test cases

## Pitfalls

| Pitfall | Fix |
|---|---|
| Only grading the final text | Check environment state and trajectory |
| Running against live systems | Sandboxes, mocked tools, recorded fixtures |
| Single run per task | Repeat runs; report pass^k and variance |
| Exact trajectory match too strict | Subset/in-order match or rubric judge |
| Ignoring cost/steps | Track efficiency, set step budgets |

## Interview Quick Answer
> I evaluate agents on outcome, trajectory and individual steps. The headline is task success, ideally verified by checking the end state in a sandbox. I also measure tool-selection and argument accuracy, trajectory match against reference paths, step count, recovery from errors, safety violations, cost and latency, and run tasks multiple times to measure consistency. In production I trace every run and feed failures back into the suite.

## Rule of Thumb
Grade what the agent **did** to the world, not just what it **said**, and run each task more than once.

## Related
- [Q5 Agent infinite tool loop](../easy_questions/ans/05_agent_infinite_tool_loop.md)
- [Q23 Agent observability](../hard_questions/ans/23_agent_observability.md)
