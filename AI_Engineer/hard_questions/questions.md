# AI Engineer — Hard Questions

[← Back to AI_Engineer](../README.md)

Questions marked **(added)** were not in the original list. I added them to fill gaps.

## LLM Serving & Inference

| # | Question | Answer |
|---|----------|--------|
| 1 | Batch LLM requests on one GPU while users wait. | [Answer](ans/01_batching_llm_requests_single_gpu.md) |
| 2 | What is the KV cache, and why does it limit concurrency? | [Answer](ans/02_kv_cache_and_concurrency.md) |
| 3 | Autoscale LLM inference on Kubernetes. Why is CPU the wrong scaling signal? | [Answer](ans/03_autoscaling_llm_inference_kubernetes.md) |
| 4 | Distillation vs quantization: what do you trade away? | [Answer](ans/04_distillation_vs_quantization.md) |
| 5 | Diagnose high latency. Which metrics matter? | [Answer](ans/05_diagnosing_high_latency.md) |
| 33 | **(added)** How does speculative decoding speed up inference, and when does it not help? | [Answer](ans/33_speculative_decoding.md) |
| 34 | **(added)** How do you cut LLM inference cost by 50%+ without hurting quality? | [Answer](ans/34_prompt_caching_cost_optimization.md) |

## RAG

| # | Question | Answer |
|---|----------|--------|
| 6 | Version documents so stale content never surfaces. | [Answer](ans/06_document_versioning_stale_content.md) |
| 7 | Ingest large tables without losing structure. | [Answer](ans/07_ingesting_large_tables.md) |
| 8 | Prompting vs RAG vs fine-tuning: how do you choose? | [Answer](ans/08_prompting_vs_rag_vs_finetuning.md) |
| 35 | **(added)** How do you choose a chunking strategy? | [Answer](ans/35_chunking_strategies.md) |
| 36 | **(added)** Why use hybrid search (BM25 + dense) and rerankers? | [Answer](ans/36_hybrid_search_reranking.md) |
| 37 | **(added)** How do you evaluate a RAG system end to end? | [Answer](ans/37_rag_evaluation_metrics.md) |
| 38 | **(added)** How do you enforce document-level permissions in multi-tenant RAG? | [Answer](ans/38_multi_tenant_rag_access_control.md) |
| 41 | **(added)** With 1M-token context windows, do we still need RAG? | [Answer](ans/41_long_context_vs_rag.md) |

## Agents & Orchestration

| # | Question | Answer |
|---|----------|--------|
| 9 | A claims-approval agent under a token budget. | [Answer](ans/09_claims_approval_agent_token_budget.md) |
| 10 | A support agent with tools, memory, and human handoff. | [Answer](ans/10_support_agent_tools_memory_handoff.md) |
| 11 | A summarizer agent in LangGraph. | [Answer](ans/11_summarizer_agent_langgraph.md) |
| 12 | Orchestrate multiple agents: planner vs executors, shared state, failure recovery. | [Answer](ans/12_multi_agent_orchestration.md) |
| 13 | Connect an agent to enterprise tools over MCP: auth, RBAC, schema-validated calls. | [Answer](ans/13_mcp_enterprise_tools_auth_rbac.md) |
| 14 | A voice calling agent: sub-second latency, interruptions, human handoff. | [Answer](ans/14_voice_calling_agent.md) |
| 15 | An AI recruiter for sales hiring: screening, outreach, and bias checks. | [Answer](ans/15_ai_recruiter_sales_hiring.md) |

## Guardrails & Responsible AI

| # | Question | Answer |
|---|----------|--------|
| 16 | Guardrails: prompt injection, PII redaction, output validation, tool permissions. | [Answer](ans/16_guardrails.md) |
| 17 | Responsible AI for regulated decisions: bias testing, explainability, audit trails. | [Answer](ans/17_responsible_ai_regulated_decisions.md) |

## Evaluation & Observability

| # | Question | Answer |
|---|----------|--------|
| 18 | How do you know it actually works? | [Answer](ans/18_how_do_you_know_it_works.md) |
| 19 | LLM-as-a-judge: failure modes and calibration. | [Answer](ans/19_llm_as_a_judge.md) |
| 20 | A model hallucinates or repeats itself on high-stakes answers. Fix it. | [Answer](ans/20_hallucination_repetition_high_stakes.md) |
| 21 | Micro vs Macro F1 on imbalanced data. | [Answer](ans/21_micro_vs_macro_f1.md) |
| 22 | Catch regressions when a prompt or model changes, and roll back safely. | [Answer](ans/22_regression_detection_rollback.md) |
| 23 | Observability for agents: traces, cost per request, alerts. | [Answer](ans/23_agent_observability.md) |

## Coding

| # | Question | Answer |
|---|----------|--------|
| 24 | A rate limiter with per-user and global limits. | [Answer](ans/24_rate_limiter_per_user_global.md) |
| 25 | Async retries with backoff, jitter, and timeouts. | [Answer](ans/25_async_retries_backoff_jitter.md) |
| 26 | A RAG pipeline over a folder of docs. | [Answer](ans/26_rag_pipeline_folder_of_docs.md) |
| 39 | **(added)** How do you get reliable structured (JSON) output from an LLM? | [Answer](ans/39_structured_outputs_reliability.md) |
| 40 | **(added)** How does LoRA/QLoRA fine-tuning work, and how do you run it? | [Answer](ans/40_lora_peft_finetuning.md) |
| 42 | **(added)** Implement a semantic cache for LLM responses. | [Answer](ans/42_semantic_cache.md) |

## System Design & FDE

| # | Question | Answer |
|---|----------|--------|
| 27 | Kafka into an AI pipeline: ordering, duplicates, consumer lag, replay. | [Answer](ans/27_kafka_ai_pipeline.md) |
| 28 | RAG over 50M patient records under HIPAA, inside a customer's VPC. | [Answer](ans/28_rag_hipaa_50m_records_vpc.md) |
| 29 | Cut an LLM search from 1.5s to under 100ms. | [Answer](ans/29_llm_search_1500ms_to_100ms.md) |
| 30 | Turn body-camera audio into a draft incident report that stays accurate and human-reviewed. | [Answer](ans/30_bodycam_audio_incident_report.md) |
| 31 | Unify fraud detection across three acquired systems. Scope the first 90 days. | [Answer](ans/31_unify_fraud_detection_90_days.md) |
| 32 | Walk through a system you owned end to end: what broke, and what you changed. | [Answer](ans/32_system_owned_end_to_end.md) |
