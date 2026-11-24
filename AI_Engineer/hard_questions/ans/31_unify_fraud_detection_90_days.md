# Q31. Unify fraud detection across three acquired systems. Scope the first 90 days.
[← Back to questions](../questions.md)

## TL;DR
- **Don't rip and replace in 90 days.** The goal for the first 90 days is a **common view of fraud** (one event schema, one entity graph, one metrics baseline) and a **unified scoring layer running in shadow** next to the three legacy systems. Production cutover comes later, per segment, behind champion/challenger.
- **Days 0-30, discovery and baseline:** stakeholder map (fraud ops, risk, compliance, product, the 3 engineering teams), an inventory of rules, models, data, labels, vendors and SLAs per system, and a **metrics baseline** (fraud loss in bps, false-positive rate, review queue volume, decision latency, chargeback rate) under *one shared definition*. Find the quick wins and the risks.
- **Days 31-60, foundations:** a **canonical event schema** and streaming ingestion from all three systems, **entity resolution** across systems (customer, device, card, account, merchant), a feature store with shared features, and a **unified label table** (chargebacks, confirmed fraud, investigator dispositions).
- **Days 61-90, shadow and first win:** a unified model or ensemble scores 100% of traffic in **shadow**. Compare with each incumbent on the same events, find the **cross-system fraud rings** that no single system could see, launch one low-risk production win (for example shared deny-lists or cross-system velocity features fed into the existing systems), and publish a roadmap with go/no-go criteria.
- **LLMs help around the core, not in it:** investigator copilot (case summarization, entity-graph narration, SAR draft assistance), rule documentation and migration, data mapping. **Core real-time scoring** stays with GBDTs, graph features and rules (latency, calibration, explainability, model risk governance).
- **Success metrics:** fraud loss bps ↓, FPR and customer-friction ↓, review queue volume and handle time ↓, cross-system ring detection ↑, with no regression on approval rate or latency SLA.

## Detailed Answer

### 1. Clarifying questions (ask before committing to a plan)
- **What are the three systems?** For example: (A) the parent's in-house card-transaction GBDT plus rules, (B) a vendor platform at an acquired fintech (Featurespace, Sardine, Sift and the like), (C) a rules-only system at an acquired marketplace. Real-time or batch? What products: cards, ACH, account opening, P2P, marketplace payouts?
- **Why unify, and what's the business driver?** Cost of three stacks, fraud rings crossing brands, regulatory pressure, customer experience. That sets the priority order.
- **Are the customer bases shared?** Overlap between the three is what makes unification valuable, since rings exploit the gaps between them.
- **Constraints:** data-sharing restrictions between legal entities (privacy notices, GLBA, GDPR if in the EU, contractual limits), vendor contract end dates, model risk management (SR 11-7 in US banking), PCI scope.
- **Who owns the decision?** The fraud strategy lead and CRO? Is there budget for headcount?
- **What can't break?** Authorization latency SLAs (for example p99 < 50 ms for card auth), approval rates, regulatory reporting (SARs).

### 2. Goals and non-goals for 90 days
**Goals**
1. A single, agreed **fraud metrics baseline** across all three businesses.
2. A **canonical event stream** and **entity graph** covering all three.
3. A unified scoring service in **shadow** on 100% of traffic for at least one product line.
4. One **production quick win** with measured impact.
5. A sequenced roadmap for consolidation, approved by risk and compliance.

**Non-goals**
- Decommissioning any system.
- Retraining every model.
- Changing customer-facing decision flows without champion/challenger evidence.

### 3. Stakeholder map
| Group | What they care about | What we need from them |
|---|---|---|
| Fraud strategy / ops leads (×3) | Losses, queue workload, rule agility | Rule inventories, pain points, label definitions |
| Investigators / analysts | Tools, case handling time | Workflow shadowing, copilot requirements |
| Risk & compliance, MRM | Model governance, fair lending/UDAAP, SAR/AML linkage | Validation requirements, approval gates |
| Legal / privacy | Cross-entity data sharing | Data-sharing approvals, DPIA |
| Engineering teams (×3) | Stability, ownership | Data access, integration points |
| Product / payments | Approval rates, friction | Thresholds for acceptable friction |
| Finance | Loss accounting, chargebacks | Loss data reconciled to the GL |
| Vendors | Contracts | Data export, contract terms |

### 4. Days 0-30: discovery and baseline

**Week 1-2: inventory**
- **Rules:** export all rules from each system with their logic, owner, creation date, hit rate, precision and the action each triggers (block, review, step-up). Typical finding: thousands of rules, 30% of them dead, and many overlapping.
- **Models:** algorithm, features, training data window, label definition, refresh cadence, performance, validation docs, explainability method.
- **Data:** event sources, schemas, latency (real-time vs batch), retention, PII fields, identity keys (email, phone, device ID, card fingerprint, bank account hash, IP).
- **Labels:** what counts as "fraud" in each system (chargeback reason codes, confirmed ATO, first-party fraud, friendly fraud), the label delay (chargebacks arrive 30-90+ days later), and investigator disposition codes.
- **Decisions and SLAs:** where in the flow each system decides (pre-auth, post-auth, onboarding, payout), its latency, and its fail-open/fail-closed behavior.
- **Ops:** queue sizes, handle time, team sizes, tooling (case management).
- **Vendors and contracts:** renewal dates, data portability.

**Week 2-4: baseline metrics under one definition**
Agreeing on definitions is often the hardest and most valuable part of the first month.

```sql
-- Canonical metric: fraud loss in basis points of volume, by system and product, monthly
SELECT
  source_system,
  product,
  date_trunc('month', txn_ts)                                         AS month,
  SUM(amount_usd)                                                     AS volume_usd,
  SUM(CASE WHEN fraud_label = 1 THEN loss_usd ELSE 0 END)             AS fraud_loss_usd,
  10000.0 * SUM(CASE WHEN fraud_label = 1 THEN loss_usd ELSE 0 END)
          / NULLIF(SUM(amount_usd), 0)                                AS fraud_loss_bps,
  -- false positive rate among actioned: declined/reviewed but not fraud
  1.0 * SUM(CASE WHEN action IN ('decline','review') AND fraud_label = 0 THEN 1 ELSE 0 END)
      / NULLIF(SUM(CASE WHEN action IN ('decline','review') THEN 1 ELSE 0 END), 0) AS actioned_fp_ratio,
  1.0 * SUM(CASE WHEN action = 'review' THEN 1 ELSE 0 END) / COUNT(*) AS review_rate
FROM canonical_decisions d
LEFT JOIN canonical_labels l USING (event_id)
WHERE txn_ts < now() - interval '90 days'      -- label maturity window
GROUP BY 1, 2, 3;
```

Metrics to baseline:
- **Fraud loss in bps** (gross and net of recoveries).
- **False positive rate / FP ratio** (good customers declined or reviewed per caught fraud), plus **insult rate**.
- **Approval rate**, and **step-up/friction rate**.
- **Review queue** volume, SLA, handle time, and investigator **hit rate**.
- **Detection latency** and **decision latency p99**.
- **Chargeback rate** (card network thresholds matter, for example monitoring programs).
- **Label maturity:** what fraction of fraud is known at T+30 / 60 / 90 days.

**Day-30 deliverables:** an inventory doc, a baseline dashboard, an overlap analysis (how many entities appear in 2+ systems), a top-10 risk list, a list of quick wins, and a draft target architecture.

### 5. Days 31-60: foundations

#### 5.1 Canonical event schema
```yaml
# canonical_event.avsc (described in YAML for readability)
name: FraudEvent
fields:
  - {name: event_id, type: string}            # globally unique
  - {name: source_system, type: {enum: [A, B, C]}}
  - {name: source_event_id, type: string}
  - {name: event_type, type: {enum: [card_auth, ach_debit, p2p_send, payout, login, account_open, profile_change]}}
  - {name: event_ts, type: timestamp-millis}
  - {name: amount_minor, type: ["null", long]}
  - {name: currency, type: ["null", string]}
  - {name: customer_ref, type: string}        # source-local id
  - {name: account_ref, type: ["null", string]}
  - {name: instrument, type: {type: record, fields: [{name: kind, type: string}, {name: fingerprint, type: string}]}}
  - {name: device, type: ["null", {type: record, fields: [{name: device_id, type: string}, {name: ip, type: string}, {name: user_agent_hash, type: string}]}]}
  - {name: counterparty_ref, type: ["null", string]}
  - {name: merchant, type: ["null", {type: record, fields: [{name: mcc, type: string}, {name: merchant_id, type: string}]}]}
  - {name: legacy_decision, type: {type: record, fields: [{name: action, type: string}, {name: score, type: ["null", double]}, {name: rules_fired, type: {type: array, items: string}}]}}
  - {name: schema_version, type: int}
```
- Each system gets an **adapter** (CDC from its DB, or tapping its event bus) that publishes canonical events to Kafka, without changing the legacy decision path.
- Include the **legacy decision** in the event. Shadow comparison needs it.
- PII is tokenized or hashed consistently (keyed HMAC) so cross-system matching works without raw data sprawl.

#### 5.2 Entity resolution across systems
- Deterministic matches: hashed email, phone (E.164), card fingerprint, bank account and routing hash, government ID hash (if permitted), device ID.
- Probabilistic matching: name + address + DOB similarity (Splink, Zingg or similar), with tuned thresholds, human review of a sample, and precision measured.
- Build an **identity graph** (nodes: person, account, device, instrument, IP, address; edges with first/last seen). Graph features like connected component size, shared devices and fraud-labelled neighbors are where cross-system rings show up.
- **Legal gate:** confirm that cross-entity data use is allowed by the privacy notices and contracts. You may need updated notices or a legitimate-interest assessment. Fraud prevention is usually a permitted purpose, but verify.

#### 5.3 Feature store
- Pick one feature platform (Feast, Tecton, Databricks/SageMaker feature store, or in-house) with **point-in-time correct** offline features and low-latency online features (Redis/DynamoDB).
- First feature sets:
  - Cross-system velocity: count and amount per entity per 1h/24h/7d **across all three systems**.
  - Graph features: shared device count, distance to known fraud, component size.
  - Behavioral features: time since account open, new device, new payee.
  - Legacy scores as features (a stacking ensemble is a fast way to get lift).

#### 5.4 Unified labels
- A `canonical_labels` table: `event_id, label, label_type (chargeback, confirmed_ato, first_party, scam), label_source, label_ts, loss_usd, recovered_usd`.
- Harmonize disposition codes across the three case-management tools with a mapping table, reviewed by the fraud leads.
- Handle **label delay**: train only on matured windows, and use early proxies (investigator confirms, customer disputes) with care.
- Handle **selection bias**: declined transactions never get labels. Use reject inference cautiously, or keep a small randomized holdout if the business accepts it.

### 6. Days 61-90: shadow scoring and the first win

```mermaid
flowchart LR
  subgraph Legacy["Legacy (unchanged decision path)"]
    A[System A<br/>GBDT + rules] --> DA[Decision A]
    B[System B<br/>Vendor] --> DB[Decision B]
    C[System C<br/>Rules] --> DC[Decision C]
  end
  A -. CDC/adapters .-> K[(Kafka: canonical FraudEvent)]
  B -. .-> K
  C -. .-> K
  K --> ER[Entity resolution +<br/>identity graph]
  K --> FS[Feature store<br/>online + offline]
  ER --> FS
  FS --> U[Unified scoring service<br/>SHADOW mode]
  U --> SL[(Shadow decisions log)]
  DA & DB & DC --> CMP[Champion vs challenger<br/>comparison + dashboards]
  SL --> CMP
  LBL[(Unified labels)] --> CMP
  U -.later: per-segment cutover.-> DA
  INV[Investigator copilot<br/>LLM over cases + graph] --- ER
```

**Unified model v0**
- A GBDT (LightGBM/XGBoost) per event family (card, ACH/P2P, onboarding) using unified features plus the legacy scores. Calibrate it (isotonic) so a single threshold policy works across segments.
- Explainability: SHAP reason codes, needed by investigators and for adverse action where it applies.
- **Shadow mode:** score every event in parallel, log the scores, take no action. Measure on the matured labels:
  - At the **same review/decline rate** as the incumbent, how much more fraud $ is caught?
  - At the **same fraud capture**, how many fewer false positives?
  - Latency p99 against the SLA.
  - Segment breakdowns (product, geography, new vs existing customers) and fairness checks.

**First production win (low risk, high signal):** pick one of these.
- **Cross-system deny/watch lists:** confirmed-fraud devices, instruments and identities from any system pushed into all three as a rule input.
- **Cross-system velocity features** exposed as an API that the legacy systems call, or as a vendor-system custom field.
- **Rule cleanup:** retire dead or redundant rules with zero precision (reduces FPs and queue load).
- **Unified investigator view:** one entity page linking cases across the three systems.

**Day-90 deliverables:** shadow results, a business case with the projected bps and FP improvements, an MRM validation plan, a cutover roadmap (by segment, with champion/challenger traffic splits such as 5% → 25% → 50%), vendor contract decisions, and an org/ownership model.

### 7. Where LLMs help and where they don't
| Use | LLM fit | Why |
|---|---|---|
| Real-time transaction scoring | **No** | Latency (ms budgets), cost per event, calibration, stability, MRM explainability. GBDT/graph models are better |
| Investigator copilot: summarize a case, timeline, linked entities, prior cases | **Yes** | Saves handle time. The human decides |
| Draft SAR narratives from structured case data | **Yes, with review** | Big time saver. Compliance must review. Cite the data |
| Rule inventory: explain or translate legacy rule DSLs into canonical form | **Yes** | Speeds up migration. Verify by replaying the rules on historical data |
| Schema mapping / data dictionary drafts | **Yes** | Accelerates adapters. Engineers validate |
| Scam detection in free text (P2P memos, chat, support tickets) | **Maybe** | Run a small classifier or embedding model as a *feature*, async or near-real-time |
| Autonomous blocking decisions | **No** | Governance, accountability, adversarial manipulation |

Copilot guardrails: read-only tools, PII access governed by the investigator's entitlements, citations to case records, logging, and no customer-facing output without review.

### 8. Architecture end state (post-90 days, for context)
- One decisioning platform: rules engine plus a unified model ensemble, a feature store and an identity graph, with per-product policies.
- Legacy systems retired segment by segment as the champion/challenger results prove out. The vendor system may be kept for niches where it outperforms.
- A unified case management tool and investigator copilot.

### 9. Risk and compliance
- **Model risk management (SR 11-7 / OCC 2011-12 in US banking):** documentation, independent validation, ongoing monitoring, and change management for the unified model before it makes production decisions.
- **Fair lending / UDAAP:** if decisions affect credit or account access, check for disparate impact and avoid prohibited bases and proxies.
- **AML linkage:** fraud and AML teams share signals. SAR obligations and timelines.
- **PCI DSS:** keep PANs out of the feature store and use fingerprints or tokens.
- **Privacy:** cross-entity sharing approvals, data minimization, retention limits.
- **Operational risk:** fail-open vs fail-closed policies, and a kill switch for any new rule or model.

### 10. Failure modes and risks in the plan
| Risk | Mitigation |
|---|---|
| Metric definitions disputed, so no baseline | Exec sponsor arbitrates. Document definitions in week 2 |
| Data access delayed by legal | Start legal review on day 1. Use aggregated or hashed data meanwhile |
| Entity resolution false merges | Precision-first thresholds, human-reviewed samples, merge confidence as a feature |
| Shadow comparison biased by label selection | Compare at matched operating points, use matured labels, holdouts |
| Legacy teams feel threatened | Involve them as owners of their adapters and segments. Credit the wins |
| Scope creep (rebuilding everything) | Fixed 90-day goals, clear non-goals, weekly steering |
| Fraudsters adapt during migration | Keep the legacy controls on, add the shared signals, monitor attack patterns |

### 11. Cadence and governance
- Weekly working group (eng + fraud ops), biweekly steering (CRO delegate, compliance, product), and a day-30/60/90 readout with go/no-go decisions.
- A RAID log (risks, assumptions, issues, dependencies).
- Decision records (ADRs) for the schema, feature store and ER approach.

### 12. Team and cost (rough)
- Core team: 1 lead (you), 2-3 data/ML engineers, 1-2 data engineers per legacy integration, 1 fraud strategy analyst, part-time privacy/legal and MRM.
- Infrastructure: Kafka and stream processing, a feature store, a graph store (Neo4j/Neptune or graph features computed in Spark), and a modest model-serving footprint. Usually small next to fraud losses (a 1 bps reduction on $10B volume = $1M/year).

## Trade-offs

| Decision | Option A | Option B | Guidance |
|---|---|---|---|
| Strategy | Rip & replace with one new system | Strangler: shared data/features → shadow → per-segment cutover | Strangler. Fraud systems are high-risk to swap |
| Model | One global model | Per-product models with shared features | Per event family at first. Merge later if the data supports it |
| Entity resolution | Deterministic only | Deterministic + probabilistic | Start deterministic (high precision), add probabilistic with confidence |
| Vendor system | Replace | Keep as a feature/score input | Use its score as a feature until the challenger beats it |
| First win | Unified model in production | Shared lists / velocity into legacy | Shared signals. Low risk, quick, shows the cross-system value |
| LLM | In the scoring path | Investigator copilot + migration tooling | Copilot. Keep scoring classical |

## Common Pitfalls
- Promising a single unified production model in 90 days.
- Comparing systems with different fraud definitions or immature labels.
- Ignoring label delay and the selection bias from declined transactions.
- Skipping legal review of cross-entity data sharing.
- Entity resolution false merges that link innocent customers to fraud.
- Forgetting latency SLAs and fail-open behavior on the authorization path.
- Treating rules as legacy junk. Many encode hard-won knowledge, so inventory them and measure each rule's precision before cutting.
- Not involving MRM and compliance until the end.
- Putting an LLM in the real-time decision path because it's fashionable.

## Likely Follow-up Questions
1. **How would you prove the unified model is better?**
   Shadow scoring on the same events, compared at matched operating points (same decline/review rate) on matured labels, in fraud $ caught and FPs. Then a champion/challenger live split with guardrails, plus segment and fairness breakdowns and MRM validation.
2. **The vendor system is a black box. What do you do?**
   Use its score as a feature, export its decisions and reason codes, negotiate data export, and plan replacement only once the challenger beats it on matched metrics. Watch the contract renewal dates.
3. **How do you handle label delay?**
   Train on windows matured past 60-90 days. Use early signals (disputes, investigator confirmations) as auxiliary labels. Monitor drift with unsupervised metrics such as score distribution and PSI in the meantime.
4. **How do you detect fraud rings across the systems?**
   Use the identity graph: connected components over shared devices, instruments and addresses, labelled-neighbor features, community detection, and analyst tooling to explore clusters. Rings are often only visible after the merge.
5. **What would make you stop or change the plan at day 60?**
   No legal approval for data sharing (fall back to per-entity models plus shared hashed deny-lists), unreliable labels (invest in label ops first), or a business shift in priorities (for example scams over card fraud).
6. **How does the investigator copilot avoid leaking data?**
   It enforces the investigator's entitlements on the tools it calls, retrieves only the case-linked entities, cites records, logs everything, uses an internal or BAA/DPA-covered model, and includes no customer-facing generation.

[← Back to questions](../questions.md)
