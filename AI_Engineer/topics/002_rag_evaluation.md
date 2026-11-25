# RAG Evaluation

## Evaluate the Two Halves Separately
A RAG answer can fail because retrieval found the wrong chunks **or** because generation misused good chunks. Measure both.

```
Question → [Retriever] → contexts → [Generator] → answer
              │                          │
     Retrieval metrics           Generation metrics
  (did we find the right info?)  (did we use it correctly?)
```

## Retrieval Metrics

| Metric | Question it answers | Needs |
|---|---|---|
| **Recall@k / Hit rate** | Is a relevant chunk in the top k? | Labeled relevant chunk IDs |
| **Precision@k** | What fraction of top k is relevant? | Labeled relevant chunk IDs |
| **MRR** (Mean Reciprocal Rank) | How high is the first relevant chunk? | Labels |
| **nDCG** | Are the most relevant chunks ranked highest? | Graded relevance labels |
| **Context precision** (Ragas) | Are relevant chunks ranked above irrelevant ones? | Question, contexts, reference |
| **Context recall** (Ragas) | Do the contexts cover everything the reference answer needs? | Contexts, reference answer |

## Generation Metrics

| Metric | Question it answers |
|---|---|
| **Faithfulness / groundedness** | Is every claim in the answer supported by the retrieved context? (hallucination check) |
| **Answer relevancy** | Does the answer address the question? |
| **Answer correctness** | Does it match the reference answer factually? |
| **Citation accuracy** | Do cited sources actually support the statement? |
| **Refusal correctness** | Does it say "I don't know" when the context lacks the answer? |

## Failure Diagnosis Matrix

| Context recall | Faithfulness | Diagnosis | Fix |
|---|---|---|---|
| Low | — | Retriever misses the info | Chunking, embeddings, hybrid search, query rewriting |
| High, low precision | — | Right info buried in noise | Reranker, smaller top-k, metadata filters |
| High | Low | Model ignores or invents beyond context | Prompt ("answer only from context"), stronger model, grounding checks |
| High | High, low correctness | Reference/data is outdated or ambiguous | Fix source documents, dataset |

## Ragas Example
```python
from ragas import evaluate, EvaluationDataset
from ragas.metrics import Faithfulness, ResponseRelevancy, LLMContextPrecisionWithReference, LLMContextRecall

dataset = EvaluationDataset.from_list([
    {
        "user_input": "What is the refund window?",
        "retrieved_contexts": ["Refunds are accepted within 30 days of purchase."],
        "response": "You can request a refund within 30 days.",
        "reference": "30 days from purchase.",
    },
])

result = evaluate(
    dataset,
    metrics=[Faithfulness(), ResponseRelevancy(), LLMContextPrecisionWithReference(), LLMContextRecall()],
    llm=evaluator_llm, embeddings=evaluator_embeddings,
)
print(result)
```

## DeepEval Example (CI gate)
```python
from deepeval import assert_test
from deepeval.test_case import LLMTestCase
from deepeval.metrics import FaithfulnessMetric, AnswerRelevancyMetric

def test_refund_question():
    tc = LLMTestCase(
        input="What is the refund window?",
        actual_output=rag.answer("What is the refund window?"),
        retrieval_context=rag.last_contexts,
    )
    assert_test(tc, [FaithfulnessMetric(threshold=0.8), AnswerRelevancyMetric(threshold=0.7)])
```
Run with `deepeval test run test_rag.py` in CI.

## Building the RAG Test Set
- Questions from real users, with **gold source chunks/documents** labeled
- Include questions whose answer is **not** in the corpus (tests refusal)
- Multi-hop questions that need several chunks
- Synthetic generation (Ragas `TestsetGenerator`) → **human review** before use
- Re-label when documents change (versioning, see Q6 on stale content)

## What to Tune, One at a Time
Chunk size/overlap → embedding model → hybrid vs dense → top-k → reranker → prompt → LLM.
Record each experiment against the same dataset and compare.

## Pitfalls
- Only scoring the final answer → you can't tell which stage broke
- Judge model weaker than the generator → unreliable faithfulness scores
- Test set built from the same chunks the system was tuned on → optimistic results
- Ignoring latency/cost of rerankers and larger top-k

## Interview Quick Answer
> I evaluate retrieval and generation separately. For retrieval I use recall@k, MRR or Ragas context precision/recall against labeled relevant chunks; for generation I use faithfulness, answer relevancy and correctness. The combination tells me where the problem is: low recall means fix retrieval, high recall but low faithfulness means fix the prompt or model. I run this with Ragas offline and DeepEval as CI gates.

## Rule of Thumb
Fix retrieval first — the generator can't use what it never sees — then fix faithfulness.

## Related
- [001 Evals fundamentals](001_evals_fundamentals.md)
- [Q7 Ragas / DeepEval](../easy_questions/ans/07_ragas_deepeval.md)
