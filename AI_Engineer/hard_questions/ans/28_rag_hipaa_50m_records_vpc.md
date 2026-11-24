# Q28. RAG over 50M patient records under HIPAA, inside a customer's VPC.
[← Back to questions](../questions.md)

## TL;DR
- **Compliance comes first.** Get a BAA with every vendor that touches PHI, including any cloud LLM endpoint. If one is used, it must be HIPAA-eligible (for example AWS Bedrock or Azure OpenAI under the cloud BAA) and reached over **PrivateLink / Private Endpoint**. Otherwise, self-host open-weight models (vLLM on GPU nodes) inside the customer VPC. **No public egress.**
- **Access control is the real design problem.** Enforce patient-level ACLs and the *minimum necessary* standard at **retrieval time** with metadata filters (care-team, facility, consent, purpose of use). Post-filtering LLM output is too late. The LLM must never see a chunk the user isn't allowed to see.
- **Capacity:** 50M records ≈ 400M-1B chunks. At 768 dims in float32 that is about 3 KB per vector, so **1.2-3 TB of raw vectors**. Use quantization (int8/PQ/binary + rescoring), shard by tenant/facility, and **narrow with structured filters first**. Most clinical questions are scoped to one patient, which turns a billion-vector search into a few-thousand-chunk search.
- **Hybrid retrieval:** structured data (FHIR: labs, meds, problems) goes through SQL/FHIR queries, not embeddings. Unstructured notes go through BM25 plus vector search. The LLM synthesizes an answer with **citations to source documents**.
- **Security:** KMS customer-managed keys for every store, TLS 1.2+ everywhere, an immutable **audit log of every access** (who, which patient, which docs, why), retention under the customer's policy, and no PHI in application logs or traces.
- **Deploy and evaluate:** Terraform + Helm into the customer's account, with no vendor access by default. Evaluate with **clinicians** (faithfulness, completeness, harmful omissions) plus automated retrieval metrics. Roll out read-only, with a clinician kept in the loop.

## Detailed Answer

### 1. Clarifying questions
- **Who are the users and what is the use case?** Clinicians asking about one patient ("summarize this patient's cardiac history"), population/cohort questions ("patients on drug X with eGFR < 30"), or coding/billing/research staff? Each has a different *purpose of use* and access scope.
- **What is a record?** An encounter, a note, or a whole chart? What mix of formats: FHIR resources, HL7v2, C-CDA, scanned PDFs (OCR), free-text notes?
- **Cloud and models:** Which cloud? Is a BAA-covered managed LLM allowed, or does policy require fully self-hosted?
- **Identity:** Which IdP (Azure AD/Okta, SMART on FHIR)? Where does the care-team relationship data live (EHR, ADT feeds)?
- **Freshness:** How quickly must new notes become searchable (minutes or nightly)?
- **Is de-identified data allowed for some use cases** (research, eval), or is everything identified PHI?
- **Is this clinical decision support?** Is any output used directly for treatment decisions? That affects FDA CDS considerations and how much human review is needed.

### 2. Requirements
**Functional**
- Natural-language Q&A over a patient's chart, with citations.
- Cohort queries that combine structured filters with note search.
- Results only for patients and documents the user is authorized to see.
- Break-the-glass flow (emergency access) with justification and extra audit.

**Non-functional**
- HIPAA Security Rule safeguards (administrative, physical, technical) and the minimum necessary standard.
- Everything inside the customer VPC with no public egress, and customer-managed keys.
- p95 under 5 s for patient-scoped Q&A, under 15 s for cohort.
- Availability 99.9% during clinical hours. Multi-AZ.
- New documents indexed within 15 minutes.
- Audit: 6-year retention for HIPAA documentation (policies, and many orgs apply it to access logs too). Follow the customer's policy and state law.

### 3. Capacity estimates
| Item | Assumption | Result |
|---|---|---|
| Records | 50M documents (notes, reports, discharge summaries) | |
| Avg size | ~1,500 tokens per document (notes vary from 200 to 10k) | ~75B tokens total |
| Chunking | 512-token chunks, 64 overlap → ~3.5 chunks/doc avg (long docs dominate) | **~175M-250M chunks**. If "records" means patients with ~20 docs each, it's ~1B docs, so plan for up to **1B chunks** |
| Vector size | 768-d float32 = 3,072 B | 250M → **0.77 TB**. 1B → **3 TB** raw |
| HNSW overhead | ~1.5x (graph links, M=16) | 1.1-4.5 TB |
| int8 scalar quantization | 4x smaller | 250M → ~190 GB in RAM, plus the full vectors on disk for rescoring |
| Binary quantization (1 bit/dim) | 32x smaller | 250M → ~24 GB. Rescore top-k with float vectors |
| Embedding compute (backfill) | 75B tokens. A self-hosted bge-base-class model at ~20k tokens/s per A10G-class GPU | ~3.75M GPU-seconds ≈ 1,040 GPU-hours, so **about 2 days on 20 GPUs** |
| Ongoing | ~100k new docs/day | Trivial, a single GPU |
| BM25 index | ~30-50% of raw text | 75B tokens ≈ 300 GB text, so ~100-150 GB inverted index |

**Key insight:** most queries are **patient-scoped**. A patient has perhaps 50-5,000 chunks. Filter by `patient_id` first and you're doing a tiny brute-force or filtered ANN search. You don't need a perfect global ANN for that case. A global ANN is only needed for cohort discovery, and even there structured filters usually narrow the set first.

### 4. Architecture

```mermaid
flowchart TB
  subgraph CustomerVPC["Customer AWS account / VPC (private subnets, no IGW egress)"]
    EHR[EHR / FHIR server<br/>HL7 feeds] --> ING[Ingestion workers<br/>parse, OCR, chunk, PHI-tag]
    ING --> EMB[Embedding service<br/>self-hosted on GPU nodes]
    EMB --> VDB[(Vector + BM25 index<br/>OpenSearch / pgvector / Milvus<br/>sharded by facility)]
    ING --> META[(Postgres: doc metadata<br/>patient_id, encounter, ACL attrs)]
    EHR --> FHIRQ[FHIR / SQL query service]
    U[Clinician UI / EHR-embedded SMART app] -->|OIDC via IdP| GW[API Gateway + AuthZ PDP<br/>OPA / Cedar]
    GW --> ORCH[RAG Orchestrator]
    ORCH -->|ACL-filtered retrieval| VDB
    ORCH --> FHIRQ
    ORCH --> RR[Reranker]
    ORCH --> LLM[LLM: vLLM self-hosted<br/>OR Bedrock/Azure OpenAI via PrivateLink]
    ORCH --> AUD[(Immutable audit log<br/>S3 Object Lock / CloudTrail Lake)]
    KMS[KMS CMKs] -.encrypts.- VDB & META & AUD & EMB
    VPCE[VPC endpoints: S3, KMS, STS, Bedrock] -.- ORCH
  end
  SIEM[Customer SIEM] <-- AUD
```

### 5. Component deep dives

#### 5.1 Model hosting options
| Option | PHI leaves the VPC? | BAA | Notes |
|---|---|---|---|
| Self-hosted open-weight (Llama/Qwen/Mistral-class via vLLM/TGI on EKS GPU nodes) | No | Not needed for the model itself (customer infrastructure) | Full control. You own GPU capacity, patching and quality. Air-gap friendly |
| AWS Bedrock via interface VPC endpoint (PrivateLink) | Traffic stays on the AWS network. Bedrock is HIPAA-eligible under the AWS BAA | Customer's AWS BAA | Check that the chosen model and region are HIPAA-eligible. Bedrock states it doesn't use prompts to train models |
| Azure OpenAI with Private Endpoint | On the Azure backbone | Microsoft BAA covers Azure OpenAI | Request the modified abuse-monitoring exemption if prompts must not be retained for human review |
| Public SaaS LLM API without a BAA | Yes | No | **Not allowed for PHI** |

The default for "inside the customer VPC" is self-hosted embeddings plus reranker (small models, cheap) and a choice between self-hosted and PrivateLink-managed for the generator. Many customers accept Bedrock or Azure OpenAI under their existing cloud BAA. Strict ones (government, some health systems) require self-hosted.

#### 5.2 Ingestion
- **Sources:** FHIR bulk export (`$export` ndjson), HL7v2 via an interface engine, document stores (PDF/TIFF → OCR with Textract (HIPAA-eligible) or self-hosted Tesseract/docTR).
- **Chunking:** section-aware. Clinical notes have headers (HPI, Assessment, Plan, Medications), so chunk by section and keep the header in each chunk. Add contextual metadata such as "Discharge Summary, 2024-03-12, Cardiology".
- **Metadata per chunk:** `patient_id`, `encounter_id`, `doc_type`, `author`, `facility_id`, `department`, `service_date`, `sensitivity_tags` (behavioral health, substance use disorder (42 CFR Part 2), HIV, reproductive health, genetic), `consent_flags`, `source_uri`, `doc_version`.
- **Sensitive categories:** 42 CFR Part 2 (SUD records) and state laws impose stricter rules than HIPAA. Tag at ingestion (from source system flags plus a classifier) and exclude by default unless the policy allows them.
- **Incremental updates:** CDC or subscription feeds (FHIR Subscriptions), with upserts keyed on `doc_id + version`. Amended or retracted notes must be tombstoned in the index.

#### 5.3 Access control (the core design)
- **AuthN:** OIDC/SAML through the customer IdP. When embedded in the EHR, a SMART on FHIR launch gives patient context and user scopes.
- **AuthZ:** ABAC with a policy decision point (OPA or AWS Verified Permissions/Cedar). Inputs: the user's role, care-team membership (from ADT/attribution data), facility, purpose of use, and the patient's consent flags.
- **Enforced in the retrieval query.** The orchestrator turns the policy into a filter that the vector/BM25 query must include. The LLM gets only authorized chunks.

```python
def build_filter(user, patient_id, purpose):
    decision = pdp.evaluate(subject=user, action="read_clinical_notes",
                            resource={"patient_id": patient_id}, context={"purpose": purpose})
    if not decision.allow:
        raise Forbidden(decision.reason)
    f = {"bool": {"filter": [
        {"term": {"patient_id": patient_id}},
        {"terms": {"facility_id": decision.allowed_facilities}},
    ], "must_not": [
        {"terms": {"sensitivity_tags": decision.excluded_sensitivities}},  # e.g. ["part2", "psychotherapy_notes"]
    ]}}
    return f
```

The OpenSearch k-NN query uses efficient filtering (the Lucene and Faiss engines support `filter` inside `knn`):

```json
{
  "size": 20,
  "query": {
    "knn": {
      "embedding": {
        "vector": [0.012, -0.03, "..."],
        "k": 50,
        "filter": { "bool": { "filter": [
          {"term": {"patient_id": "P123"}},
          {"terms": {"facility_id": ["F1","F7"]}}
        ], "must_not": [{"terms": {"sensitivity_tags": ["part2"]}}]}}
      }
    }
  }
}
```

- **Cohort queries:** the user's population scope (for example "patients attributed to my clinic") comes from a structured query first. The result is a patient-ID set used as a filter, or the query runs in an approved research context over **de-identified** data.
- **Break-the-glass:** a justification is required, access is time-boxed and logged with high severity, and privacy officers review it.
- **Cache and re-check:** semantic caches must be keyed by user and authorization scope, or skipped entirely for PHI. Never share cached answers across users.

#### 5.4 Hybrid retrieval with structured data
- Many clinical questions are **structured**: "latest HbA1c", "current meds", "allergies". Answer them from FHIR (`Observation?patient=P123&code=4548-4&_sort=-date&_count=1`) or a SQL warehouse (OMOP), not by embedding search, which can miss values or pull stale ones.
- **Query router:** a small classifier or LLM tool-call decides among `fhir_query`, `note_search` (BM25 + vector) and `both`.
- **Fusion:** Reciprocal Rank Fusion of BM25 and vector results, then a cross-encoder rerank (self-hosted, for example a BGE reranker) over the top 50.
- BM25 matters a lot in clinical text: drug names, abbreviations, codes (ICD-10, LOINC) and lab names are exact-match-heavy.
- **Temporal grounding:** include dates in the chunks and the prompt, and sort evidence chronologically for summaries.

#### 5.5 Generation
- Prompt: "Answer using only the provided sources. Cite [doc_id:section:date]. If the information isn't in the sources, say so. Do not give diagnoses or treatment recommendations beyond what is documented."
- Structured output: an answer plus a list of `{claim, citations}`. Validate that each citation exists in the retrieved set and drop or flag claims without one.
- The UI shows citations as clickable links to the source note in the EHR, so clinicians can verify.

#### 5.6 Vector DB choice and sharding
| Option | Pros | Cons |
|---|---|---|
| **OpenSearch** (self-managed or the AWS managed service in-VPC) | BM25 + k-NN in one engine, efficient filtering, FGAC, KMS encryption, HIPAA-eligible service | Tuning heavy at the billion scale. JVM ops |
| **pgvector** (Aurora/RDS Postgres) | ACL joins in SQL, transactional, team familiarity, HIPAA-eligible | Billion-scale HNSW is hard. Fine if queries are always patient-scoped (a B-tree on `patient_id` plus exact distance) |
| **Milvus / Qdrant / Weaviate** self-hosted on EKS | Built for billion scale, quantization, payload filters | More ops. You own HA, backups and encryption config |

**Sharding:**
- Route by `facility_id` or `patient_id` hash (custom routing in OpenSearch), so patient-scoped queries hit **one shard**.
- Use separate indices for sensitive categories so they can be excluded and audited cleanly.
- For cohort ANN, scatter-gather across shards with a pre-filter.
- Practical recommendation: **pgvector or OpenSearch for patient-scoped search** (small per-patient candidate sets, exact search is fine), plus an OpenSearch/Milvus global index with quantization only if cohort semantic search is truly required.

#### 5.7 De-identification
- For eval datasets, analytics and any vendor-side debugging, use **Safe Harbor** (remove the 18 identifiers) or **Expert Determination**.
- Tools: Microsoft Presidio, Philter, AWS Comprehend Medical `DetectPHI`, or a fine-tuned clinical NER model. Measure **recall** on identifiers, since a missed name is a breach.
- Production Q&A for treatment generally uses identified PHI (treatment is a permitted use). De-identification protects secondary uses.
- Logs and traces: redact or hash PHI. Keep prompt/response bodies only in the encrypted audit store, not in Datadog, Splunk or similar unless that vendor is under a BAA and configured for it.

### 6. Data model
```sql
CREATE TABLE chunk (
  chunk_id        UUID PRIMARY KEY,
  doc_id          TEXT NOT NULL,
  doc_version     INT NOT NULL,
  patient_id      TEXT NOT NULL,
  encounter_id    TEXT,
  facility_id     TEXT NOT NULL,
  doc_type        TEXT NOT NULL,          -- discharge_summary, progress_note, radiology...
  service_date    DATE NOT NULL,
  section         TEXT,
  sensitivity     TEXT[] DEFAULT '{}',    -- part2, behavioral_health, hiv, genetic
  text_enc        BYTEA NOT NULL,         -- app-level envelope encryption optional
  embedding       VECTOR(768),
  is_current      BOOLEAN DEFAULT TRUE,
  ingested_at     TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON chunk (patient_id, service_date DESC) WHERE is_current;
CREATE INDEX ON chunk USING hnsw (embedding vector_cosine_ops);  -- only if global search needed

CREATE TABLE access_audit (
  audit_id     UUID PRIMARY KEY,
  ts           TIMESTAMPTZ NOT NULL,
  user_id      TEXT NOT NULL,
  user_role    TEXT NOT NULL,
  patient_id   TEXT,
  purpose      TEXT NOT NULL,             -- treatment, payment, operations, research
  query_hash   TEXT NOT NULL,             -- query text stored encrypted separately
  chunk_ids    UUID[] NOT NULL,           -- exactly what was shown to the LLM
  decision     TEXT NOT NULL,             -- allow, deny, break_glass
  model        TEXT, prompt_version TEXT
);
```
Ship the audit log to **S3 with Object Lock (compliance mode)** and/or CloudTrail Lake, hash-chained, and stream it to the customer's SIEM. Alert on anomalies: a user touching many patients, VIP-record access, after-hours bulk access.

### 7. Security and compliance checklist
- **BAA chain:** the customer (covered entity) signs with you (business associate) and with the cloud provider. Every subprocessor touching PHI, including observability, error tracking and support tooling, needs one or must be kept away from PHI.
- **Encryption:** KMS CMKs (customer-owned, rotation enabled) for EBS, S3, RDS/Aurora, OpenSearch and EKS secrets. TLS 1.2+ everywhere, mTLS inside the mesh (Istio/Linkerd). The GPU node local disks holding model caches are encrypted too.
- **Network:** private subnets only, no NAT or internet egress for PHI workloads. Use VPC endpoints for S3, KMS, STS, ECR, CloudWatch and Bedrock. Pull container images from in-account ECR, mirrored and scanned. Egress firewall/Network Firewall default deny.
- **IAM:** least privilege, IRSA per service, no long-lived keys. Vendor access only through a customer-approved, time-boxed role (break-glass for support), with session recording.
- **Prompt injection:** notes may contain arbitrary text (patient-provided content, faxed documents). Treat retrieved text as data. Give the model no tools that write or send externally, and validate citations.
- **Retention and deletion:** follow the customer's records policy. Index deletion must propagate (tombstone, compact, remove from caches). Backups are encrypted and expire on schedule.
- **Risk analysis:** a documented HIPAA security risk assessment, pen test, and threat model. Map controls to HITRUST/SOC 2 if the customer requires it.

### 8. Deployment into the customer account
```hcl
# Terraform (sketch) — module consumed by customer, applied in their account
module "rag_platform" {
  source              = "git::https://example.com/rag-hipaa//terraform?ref=v1.8.0"
  vpc_id              = var.vpc_id
  private_subnet_ids  = var.private_subnet_ids
  kms_key_arn         = aws_kms_key.phi.arn
  enable_bedrock_vpce = true
  gpu_node_instance   = "g5.2xlarge"
  gpu_node_min        = 2
  gpu_node_max        = 8
  opensearch_instance = "r6g.2xlarge.search"
  opensearch_nodes    = 6
  audit_bucket_object_lock_days = 2190   # ~6 years
}

resource "aws_vpc_endpoint" "bedrock_runtime" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.bedrock-runtime"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  private_dns_enabled = true
  security_group_ids  = [aws_security_group.vpce.id]
}
```
```yaml
# Helm values (excerpt)
llm:
  mode: selfhosted            # or bedrock
  vllm:
    model: /models/llama-3.1-70b-instruct-awq
    tensorParallelSize: 2
    resources: { limits: { nvidia.com/gpu: 2 } }
embeddings:
  model: /models/bge-base-en-v1.5
networkPolicy:
  defaultDenyEgress: true
audit:
  sink: s3://customer-phi-audit
  includeChunkIds: true
logging:
  redactPHI: true
```
- Model weights are delivered as signed artifacts into the customer's S3/ECR. There is no runtime download from Hugging Face.
- Upgrades are versioned releases with a changelog, applied by the customer or by you through an approved pipeline.
- The vendor's telemetry is limited to **non-PHI** metrics (latency, counts) via an approved channel, or none at all.

### 9. Evaluation
- **Golden set built with clinicians:** 300-500 questions across specialties and question types (lookup, temporal, summary, cohort), using *de-identified* or synthetic charts (Synthea) for development and an IRB- or privacy-approved sample for final validation.
- **Retrieval metrics:** recall@k for the evidence clinicians marked as relevant, plus MRR.
- **Generation metrics:** faithfulness (every claim supported by a cited source), completeness (did it miss a critical allergy or med?), harmful omission rate, citation precision. Use LLM-as-judge for scale, **calibrated against clinician ratings** (measure agreement).
- **Safety tests:** ACL red-team (try to retrieve another patient's data through prompt tricks), prompt injection inside notes, and Part 2 leakage tests. These **must be 0**.
- **Online:** thumbs and flags in the UI, a "source didn't support this" report button, weekly clinician review of a sample.
- **Bias and coverage:** quality per note type, facility and language, and OCR-heavy versus born-digital documents.

### 10. Failure modes
| Failure | Impact | Mitigation |
|---|---|---|
| ACL filter bug | PHI exposure = breach | Policy unit tests, filter injected by middleware rather than app code, canary queries in prod that assert denial |
| Stale or amended note retrieved | Wrong clinical info | Versioning, tombstones, `is_current` filter, show dates |
| Missing structured data via vectors | Wrong lab value | Router sends structured questions to FHIR |
| Hallucinated med or dose | Patient safety | Citation validation, "not found" behavior, UI disclaimers, clinician verification |
| GPU capacity exhausted | Latency, outage | Autoscale GPU nodes, queue, fallback to a smaller model, Bedrock as a secondary |
| PHI in logs | Compliance incident | Redaction middleware, log scanning, BAA-only log sinks |
| OCR garbage | Poor retrieval | OCR confidence scores, flag low-quality docs |

### 11. Rollout plan
1. **Phase 0 (weeks 0-4):** BAA, security review, threat model, Terraform into a non-prod account, synthetic data.
2. **Phase 1:** patient-scoped Q&A, read-only, for a pilot of 20 clinicians in one department. Citations mandatory. Thumbs feedback.
3. **Phase 2:** expand to more departments, add structured FHIR routing and cohort search for approved roles.
4. **Phase 3:** EHR-embedded SMART app, broader rollout, ongoing eval and monitoring. Go/no-go gates on faithfulness, ACL tests and clinician satisfaction.

### 12. Cost (rough)
- Embedding backfill: about 1,000 GPU-hours ≈ $1-2k on-demand (one-off).
- Serving a 70B AWQ model on 2x A100/H100-class or 4x L40S: roughly $10-25k/month per replica set, with 2-3 replicas for HA. Bedrock pay-per-token may be cheaper at low volume. Self-hosting wins at high, steady volume or when policy requires it.
- OpenSearch cluster for about 250M vectors with quantization: mid five figures per year. Patient-scoped pgvector can be much cheaper.
- The big lever is **patient-scoped retrieval**, which keeps indexes small and LLM contexts short.

## Trade-offs

| Decision | Option A | Option B | Guidance |
|---|---|---|---|
| LLM hosting | Self-hosted in VPC | Bedrock/Azure OpenAI via PrivateLink under BAA | Managed when the customer accepts the cloud BAA (faster, better models). Self-hosted for air-gap or strict policy |
| ACL enforcement | Pre-filter in retrieval | Post-filter results | **Always pre-filter.** Post-filter leaks into the LLM context and hurts recall |
| Index topology | One global index | Per-patient / per-facility partitioning | Partition, since most queries are patient-scoped |
| Vector precision | float32 | int8/binary + rescoring | Quantize at this scale. Rescore to keep recall |
| Structured data | Embed everything | FHIR/SQL tools | Use tools for values (labs, meds). Embeddings for narrative |
| De-identification | De-id everything | Identified PHI for treatment, de-id for secondary use | Treatment use permits PHI, and de-id would hurt clinical utility |
| Audit granularity | Query-level | Chunk-level (what the LLM saw) | Chunk-level. Needed for breach investigation |

## Common Pitfalls
- Calling a public LLM API without a BAA "just for the prototype" with real PHI.
- Filtering by ACL *after* generation, or relying on the prompt ("don't reveal other patients").
- Sending PHI to observability or error-tracking SaaS that isn't under a BAA.
- Embedding lab values and expecting accurate numbers. Use structured queries.
- Ignoring 42 CFR Part 2 and state-specific sensitive categories.
- A shared semantic cache across users, which leaks one user's answers to another.
- No handling for amended or deleted notes.
- Evaluating only with engineers. Clinicians have to define "correct" and "harmful".
- Downloading model weights from the internet at runtime in a no-egress VPC (the deploy breaks, and it's a supply-chain risk).
- Assuming encryption alone means compliance. Access control, audit, BAAs and risk analysis are also required.

## Likely Follow-up Questions
1. **How do you make sure the LLM provider doesn't retain PHI?**
   Use BAA-covered services with zero or limited data retention (Bedrock doesn't store prompts for training; for Azure OpenAI, apply for the abuse-monitoring opt-out). Or self-host. Verify through contract terms and architecture: PrivateLink, no other egress.
2. **A doctor asks about a patient not on their care team.**
   The PDP denies the request. The UI offers break-the-glass with a justification, which is logged and reviewed. The retrieval filter never includes that patient's chunks without an allow decision.
3. **How would you handle 1B chunks for cohort search?**
   Run structured pre-filters first (diagnosis codes, demographics) to get candidate patients, then filtered ANN on sharded, quantized indexes (IVF-PQ or HNSW + int8) with rescoring. Often cohort questions are better answered by SQL on OMOP plus text search over the narrowed set.
4. **How do you evaluate without exposing PHI to your team?**
   Synthetic data (Synthea) and de-identified samples for development. Final validation runs inside the customer environment with customer clinicians, and the vendor sees only aggregate metrics.
5. **Is this a medical device?**
   It depends on intended use. Retrieving and summarizing documented information for a clinician who can independently review the sources usually falls under the non-device CDS criteria. Making diagnostic or treatment recommendations can trigger FDA oversight. Involve regulatory counsel and design the product to show its sources.
6. **How do you handle a request to delete a patient's data?**
   HIPAA has no general right to deletion, since medical records retention rules apply. But if a record is removed or amended at the source, propagate tombstones to the index, caches and derived stores. Audit logs stay under retention.

[← Back to questions](../questions.md)
