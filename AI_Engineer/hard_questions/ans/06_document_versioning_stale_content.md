# Q6. Version documents so stale content never surfaces.
[← Back to questions](../questions.md)

## TL;DR
- Give every chunk a stable identity (`doc_id`, `chunk_id`), plus a `version`, a `content_hash`, and validity metadata (`is_current`, `valid_from`, `valid_to`). Never rely on the embedding text alone to tell you what is current.
- **Filter at query time** (`is_current = true`, or `valid_from <= now < valid_to`) so a mistake during ingestion cannot leak old content. Delete old versions asynchronously for hygiene, but don't depend on deletion for correctness.
- Make updates **atomic**: write the new version, flip the pointer (`is_current` / alias / version table), then tombstone or delete the old one. For full reindexes, build a new index and swap an alias (blue/green).
- Drive ingestion from **change events** (CDC, webhooks, `updated_at` watermarks) and send **deletes as tombstones**. A missed delete is the most common way stale data leaks.
- Invalidate **every derived layer**: semantic caches, answer caches, summaries, knowledge-graph facts, and fine-tuning snapshots.
- Prove it with **freshness evals** (questions whose answer changed) and **cite the version or date** in answers so users and auditors can check.

## Detailed Answer

### 1. Why stale content surfaces in the first place
A vector index is a derived copy of the source of truth, and the copy drifts. Common causes:

| Cause | Example |
|---|---|
| Update appended, old not removed | Policy v3 re-ingested; v2 chunks still in index |
| Chunk count changes between versions | v2 had 14 chunks, v3 has 11, and chunks 12–14 of v2 are orphaned |
| Deletes never propagated | Doc removed in SharePoint but the connector only listens for upserts |
| Non-deterministic chunk IDs | Random UUIDs per ingest, so an upsert creates duplicates instead of replacing |
| Derived caches | Semantic cache returns last week's answer |
| Duplicates across sources | Same PDF in Confluence and Google Drive, one copy updated |
| Embedding model change | Partial re-embed mixes vector spaces |

Old and new versions are semantically near-identical, so they embed almost on top of each other. Similarity search cannot tell them apart, and the old one often wins. **Freshness has to be enforced with metadata, not with similarity.**

### 2. The data model

```
documents (source of truth registry)
  doc_id         TEXT  -- stable: source system + native id, e.g. "confluence:12345"
  version        INT   -- monotonically increasing (or source revision id)
  content_hash   TEXT  -- sha256 of normalized content
  is_current     BOOL
  valid_from     TIMESTAMPTZ
  valid_to       TIMESTAMPTZ NULL   -- NULL = open-ended
  deleted        BOOL  -- tombstone
  source_updated_at TIMESTAMPTZ
  ingested_at    TIMESTAMPTZ

chunks (vector store rows)
  chunk_id       TEXT  -- deterministic: f"{doc_id}:{version}:{chunk_index}"
  doc_id, version, content_hash, is_current, valid_from, valid_to
  embedding      VECTOR
  text, title, url, acl, embedding_model
```

Key design decisions:
- **Deterministic IDs.** `chunk_id = doc_id:version:idx` makes ingestion idempotent, so a retry overwrites the same rows instead of duplicating them.
- **Content hash.** Skip re-embedding when the hash is unchanged. Many "updates" are just metadata touches, so this saves embedding cost.
- **Copy version metadata onto every chunk.** The filter runs inside the vector DB and cannot join to the documents table.
- **Bitemporal (optional).** For regulated domains, keep both *valid time* (when the policy applied) and *transaction time* (when we learned it). Then you can answer "what did the bot know on March 3?" and also "what was the policy on March 3?"

### 3. Update lifecycle (atomic swap)

```mermaid
sequenceDiagram
    participant Src as Source (CDC / webhook)
    participant Ing as Ingestion worker
    participant VS as Vector store
    participant Reg as Version registry
    participant Cache as Caches
    Src->>Ing: doc updated (doc_id, rev)
    Ing->>Ing: normalize, hash; if hash unchanged -> stop
    Ing->>VS: upsert chunks v(n+1) with is_current=false
    Ing->>Reg: tx: set v(n).is_current=false, valid_to=now; v(n+1).is_current=true, valid_from=now
    Ing->>VS: flip is_current on chunks (v(n)->false, v(n+1)->true)
    Ing->>Cache: invalidate keys tagged doc_id
    Ing->>VS: async: delete v(n) chunks after grace period
```

Order matters:
1. **Write new first, hidden** (`is_current=false`). Readers never see a half-written version.
2. **Flip** in as few operations as possible. In Postgres/pgvector this is a single transaction. In vector DBs without transactions, either flip in a tight batch, or put the "current version" pointer in a small registry and filter by `version IN current_versions`.
3. **Delete old later.** Rollback stays possible and in-flight queries don't break.

The alternative, delete-then-insert, leaves a window where the document doesn't exist and retrieval returns nothing (or something worse). Upsert-then-delete leaves a brief window where both exist, and the `is_current` filter hides that window.

### 4. pgvector implementation (transactional, simplest correct design)

```sql
CREATE TABLE chunks (
  chunk_id      TEXT PRIMARY KEY,
  doc_id        TEXT NOT NULL,
  version       INT  NOT NULL,
  content_hash  TEXT NOT NULL,
  is_current    BOOLEAN NOT NULL DEFAULT false,
  valid_from    TIMESTAMPTZ,
  valid_to      TIMESTAMPTZ,
  text          TEXT NOT NULL,
  embedding     vector(1536) NOT NULL
);
CREATE INDEX ON chunks (doc_id, is_current);
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops);
-- Partial index keeps ANN focused on live rows
CREATE INDEX chunks_current_hnsw ON chunks USING hnsw (embedding vector_cosine_ops)
  WHERE is_current;
```

```python
import hashlib
import psycopg
from datetime import datetime, timezone

def content_hash(text: str) -> str:
    normalized = " ".join(text.split()).lower()
    return hashlib.sha256(normalized.encode()).hexdigest()

def upsert_document(conn: psycopg.Connection, doc_id: str, text: str,
                    chunker, embed) -> None:
    h = content_hash(text)
    with conn.transaction():
        row = conn.execute(
            "SELECT version, content_hash FROM chunks "
            "WHERE doc_id=%s AND is_current LIMIT 1", (doc_id,)
        ).fetchone()
        if row and row[1] == h:
            return  # no-op: unchanged content
        new_version = (row[0] + 1) if row else 1
        now = datetime.now(timezone.utc)

        pieces = chunker(text)
        vectors = embed(pieces)  # do this BEFORE opening the tx in real code
        for i, (piece, vec) in enumerate(zip(pieces, vectors)):
            conn.execute(
                """INSERT INTO chunks (chunk_id, doc_id, version, content_hash,
                                       is_current, valid_from, text, embedding)
                   VALUES (%s,%s,%s,%s,false,%s,%s,%s)
                   ON CONFLICT (chunk_id) DO UPDATE
                     SET text=EXCLUDED.text, embedding=EXCLUDED.embedding""",
                (f"{doc_id}:{new_version}:{i}", doc_id, new_version, h, now,
                 piece, str(vec)),
            )
        # atomic flip
        conn.execute(
            "UPDATE chunks SET is_current=false, valid_to=%s "
            "WHERE doc_id=%s AND is_current", (now, doc_id))
        conn.execute(
            "UPDATE chunks SET is_current=true "
            "WHERE doc_id=%s AND version=%s", (doc_id, new_version))

def tombstone_document(conn, doc_id: str) -> None:
    with conn.transaction():
        conn.execute(
            "UPDATE chunks SET is_current=false, valid_to=now() "
            "WHERE doc_id=%s AND is_current", (doc_id,))
```

Query, always filtered:

```sql
SELECT chunk_id, doc_id, version, text
FROM chunks
WHERE is_current
ORDER BY embedding <=> %(qvec)s
LIMIT 8;

-- point-in-time ("what did the policy say on 2025-03-01?")
SELECT ... FROM chunks
WHERE valid_from <= '2025-03-01' AND (valid_to IS NULL OR valid_to > '2025-03-01')
ORDER BY embedding <=> %(qvec)s LIMIT 8;
```

### 5. Vector DBs without transactions (Qdrant / Pinecone / etc.)
- Use deterministic point IDs (UUIDv5 of `doc_id:version:idx`) so retries are idempotent.
- Filter on a payload index (`is_current`), or keep a **current-version registry** (Redis/Postgres) and pass `doc_id:version` pairs in the filter. The registry flip is then your atomic step.
- Delete by filter (`doc_id == X AND version < N`) as a background job.

```python
from qdrant_client import QdrantClient, models

client = QdrantClient(url="http://localhost:6333")

def search_current(qvec, tenant: str, k: int = 8):
    return client.query_points(
        collection_name="kb",
        query=qvec,
        query_filter=models.Filter(must=[
            models.FieldCondition(key="is_current", match=models.MatchValue(value=True)),
            models.FieldCondition(key="tenant", match=models.MatchValue(value=tenant)),
        ]),
        limit=k,
    ).points

def retire_old_versions(doc_id: str, current_version: int):
    client.delete(
        collection_name="kb",
        points_selector=models.FilterSelector(filter=models.Filter(must=[
            models.FieldCondition(key="doc_id", match=models.MatchValue(value=doc_id)),
            models.FieldCondition(key="version", range=models.Range(lt=current_version)),
        ])),
    )
```

LangChain's **Indexing API** (`langchain.indexes.index` plus a `SQLRecordManager`) implements the hash/record-manager pattern for you. With `cleanup="incremental"` it deletes chunks that belonged to a source but are no longer produced. With `cleanup="full"` it also removes sources that disappeared from the batch:

```python
from langchain.indexes import SQLRecordManager, index

record_manager = SQLRecordManager("pg/kb", db_url="postgresql+psycopg://...")
record_manager.create_schema()

result = index(
    docs,                      # Documents with metadata["source"] = doc_id
    record_manager,
    vector_store,
    cleanup="incremental",
    source_id_key="source",
)
print(result)  # {'num_added': .., 'num_updated': .., 'num_skipped': .., 'num_deleted': ..}
```

### 6. Change detection: CDC, webhooks, and tombstones
| Method | Latency | Catches deletes? | Notes |
|---|---|---|---|
| Full re-crawl + diff | hours | Yes (set difference) | Expensive but a good nightly reconciliation |
| `updated_at` watermark polling | minutes | **No**, unless soft deletes exist | Most common, and most common source of leaks |
| Webhooks (Confluence, SharePoint, Drive) | seconds | Usually | Can be dropped, so pair with reconciliation |
| DB CDC (Debezium → Kafka) | seconds | Yes (delete events) | Best for DB-backed content |

**Rule:** use event-driven ingestion for latency and a periodic full reconciliation for correctness. The reconciliation job lists all source IDs, compares them to indexed IDs, and tombstones the orphans.

Tombstones are explicit "this doc_id is gone as of T" records. They let downstream systems (caches, replicas, eval snapshots) learn about deletions, which a silent hard delete never tells them.

### 7. Blue/green indexes for full rebuilds
When you change the embedding model, the chunking strategy, or the schema, never mutate in place:

```
          alias "kb_live"
               │
   ┌───────────┴───────────┐
kb_v7 (blue, serving)   kb_v8 (green, building)
                         ├─ backfill all docs
                         ├─ replay CDC events since backfill start
                         ├─ run retrieval eval: must be >= blue
                         └─ swap alias -> kb_v8; keep kb_v7 for rollback
```

Elasticsearch/OpenSearch have native aliases, and Qdrant has collection aliases. For pgvector, use a view or a config pointer. **Replay the change log** that accumulated during the backfill, or the new index starts out stale.

### 8. Invalidate every derived layer
Stale content often comes from things built *from* the index rather than from the index itself:
- **Semantic / answer cache:** tag entries with the `doc_id`s (and versions) of the sources they used, and evict by tag on update. Also apply a TTL.
- **Summaries / doc cards / multi-vector representations:** regenerate them with the same version key.
- **Knowledge graph facts:** carry `source_doc_id` + `version` on each edge.
- **Agent memory / conversation history:** a long chat may hold an old chunk in context, so re-validate on follow-ups when practical.
- **Fine-tuned models:** they can't be invalidated, which is one reason to keep volatile facts out of fine-tuning.

```python
# cache entry stores provenance so it can be invalidated by doc
cache.set(key=query_hash, value={"answer": ans,
          "sources": [(c.doc_id, c.version) for c in ctx]}, ttl=3600)
# on doc update:
for key in cache_index.keys_for_doc(doc_id):
    cache.delete(key)
```

### 9. Conflicting versions that are both "current"
Sometimes both versions are legitimately valid: a regional policy, product v1 vs v2 docs, or a draft vs a published version. Handle this with **dimensions**, not with `is_current` alone: `product_version`, `region`, `status=published`. Infer the dimension from the query or the user profile and filter on it. If it's ambiguous, retrieve both and ask the model to state the difference explicitly.

At ranking time you can also add a **recency prior** (`score = sim * (1 + α·recency)`). Treat it as a tiebreaker only, never as the freshness guarantee.

### 10. Freshness evaluation and citations
- Build a **freshness test set**: questions whose answers changed (old price → new price). Assert that the answer contains the new value and *not* the old one.
- **Canary docs:** ingest a synthetic doc, update it, delete it, and check retrieval at each step in CI or as a synthetic monitor.
- Track **index lag** (`now - source_updated_at` of the latest event processed) and **orphan count** (indexed doc_ids not in source).
- **Cite version/date** in answers: "Per *Travel Policy v3 (effective 2025-09-01)*...". Users can then spot staleness, and audit logs record exactly what the model saw.

## Common Pitfalls
- Random chunk IDs, so re-ingest duplicates instead of replacing.
- Upsert-only connectors that never process deletes.
- Re-chunking produces fewer chunks, and the tail chunks of the old version are orphaned.
- Relying on post-retrieval filtering: top-k is full of stale chunks, and after filtering you have 1 result left.
- Deleting old content *before* the new version is written, which creates an empty window.
- Forgetting the semantic cache, summaries, or the BM25 index (hybrid search has **two** indexes to keep in sync).
- Partial re-embedding with a new model, which mixes vector spaces.
- Using `updated_at` from the ingestion time rather than the source's effective date.

## Likely Follow-up Questions
1. **How do you do point-in-time queries ("what was the policy last year")?** Keep old versions with `valid_from/valid_to` instead of deleting them, and filter by the requested date. Default to `is_current` otherwise.
2. **What if the vector DB has no transactions?** Use a version registry as the atomic pointer, idempotent deterministic IDs, a filter on the pointer, and async cleanup.
3. **How do you detect deletes from a source with no delete events?** Run a periodic full listing and diff it against indexed doc_ids, then tombstone the orphans.
4. **How do you keep BM25 and vector indexes consistent?** Use the same ingestion transaction or outbox pattern, the same doc_id/version keys, and the same filter applied to both retrievers before fusion.
5. **How do you avoid re-embedding cost on every update?** Hash at chunk level, and re-embed only chunks whose hash changed. Reuse vectors for unchanged chunks under the new version.
6. **How do you know staleness is not happening in production?** Run freshness canaries, monitor index lag, count orphans, track user feedback ("outdated"), and sample citations for version checks.

[← Back to questions](../questions.md)
