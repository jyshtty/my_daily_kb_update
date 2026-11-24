# Q13. Where is checkpointed state stored, and how is it retrieved?

[← Back to questions](../questions.md)

## TL;DR

- State is stored by whichever **checkpointer** you pass to `compile(checkpointer=...)`. Options are **in memory** (`InMemorySaver`), a **SQLite** file (`SqliteSaver`), **Postgres** (`PostgresSaver`), or Redis/Mongo/etc. through partner packages. On **LangGraph Platform** it is a managed Postgres database.
- Each checkpoint is keyed by **`(thread_id, checkpoint_ns, checkpoint_id)`**. Postgres uses tables such as `checkpoints`, `checkpoint_blobs`, `checkpoint_writes` (plus `checkpoint_migrations`).
- Values are serialised by a **serializer**. The default is `JsonPlusSerializer` (JSON/msgpack, with support for LangChain messages, Pydantic models, dataclasses). `EncryptedSerializer` adds encryption at rest.
- Retrieval: `graph.get_state(config)` returns the latest or a specific checkpoint, `graph.get_state_history(config)` returns all checkpoints newest-first, and adding `checkpoint_id` to the config targets one checkpoint. At a lower level there are `checkpointer.get_tuple()` and `checkpointer.list()`.
- Use the **async savers** (`AsyncPostgresSaver`, `AsyncSqliteSaver`) with `ainvoke`/`astream` and `aget_state`.

## Detailed Answer

### 1. The checkpointer interface

Every backend implements `BaseCheckpointSaver`:

| Method | Purpose |
|---|---|
| `put(config, checkpoint, metadata, new_versions)` | Save a checkpoint after a super-step |
| `put_writes(config, writes, task_id)` | Save pending writes from a finished task |
| `get_tuple(config)` | Fetch one `CheckpointTuple` (latest or by id) |
| `list(config, filter=None, before=None, limit=None)` | Iterate checkpoints |
| `delete_thread(thread_id)` | Remove a thread's data |
| `aput`, `aget_tuple`, `alist`, ... | Async equivalents |

The graph never touches the DB directly. It only calls these methods.

### 2. Backends

```python
# 1) In memory (dev/test)
from langgraph.checkpoint.memory import InMemorySaver
checkpointer = InMemorySaver()

# 2) SQLite  (pip install langgraph-checkpoint-sqlite)
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver
checkpointer = SqliteSaver(sqlite3.connect("checkpoints.db", check_same_thread=False))
# or: with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer: ...

# 3) Postgres  (pip install langgraph-checkpoint-postgres)
from langgraph.checkpoint.postgres import PostgresSaver
DB_URI = "postgresql://user:pass@localhost:5432/app?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()            # creates tables / runs migrations (first time)
    graph = builder.compile(checkpointer=checkpointer)
    graph.invoke({"messages": [("user", "hi")]},
                 {"configurable": {"thread_id": "t-1"}})
```

Async Postgres with a connection pool, typical inside FastAPI:

```python
from psycopg_pool import AsyncConnectionPool
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

pool = AsyncConnectionPool(conninfo=DB_URI, max_size=20,
                           kwargs={"autocommit": True, "prepare_threshold": 0})
checkpointer = AsyncPostgresSaver(pool)
await checkpointer.setup()
graph = builder.compile(checkpointer=checkpointer)
result = await graph.ainvoke(inputs, {"configurable": {"thread_id": "t-1"}})
```

Note: if you hand Postgres connections to the saver yourself, they need `autocommit=True` and `row_factory=dict_row`. `from_conn_string` sets this up for you.

### 3. What the Postgres schema looks like (conceptually)

```
checkpoints
  thread_id | checkpoint_ns | checkpoint_id | parent_checkpoint_id | type | checkpoint (jsonb) | metadata (jsonb)
  PK (thread_id, checkpoint_ns, checkpoint_id)

checkpoint_blobs            -- large / binary channel values, versioned
  thread_id | checkpoint_ns | channel | version | type | blob (bytea)

checkpoint_writes           -- pending writes per task
  thread_id | checkpoint_ns | checkpoint_id | task_id | idx | channel | type | blob

checkpoint_migrations       -- schema version tracking
```

Design detail: channel values are stored **per version** in `checkpoint_blobs`. A checkpoint only references the versions it uses, so a channel that did not change is not duplicated in each checkpoint.

### 4. Serialisation

```python
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer

serde = EncryptedSerializer.from_pycryptodome_aes()   # reads LANGGRAPH_AES_KEY env var
checkpointer = PostgresSaver(conn, serde=serde)
```

- `JsonPlusSerializer` handles primitives, LangChain `BaseMessage`s, Pydantic models, dataclasses, datetimes, enums, etc. It can be configured to fall back to pickle for exotic types (`pickle_fallback=True`).
- Keep state **serialisable**. Do not store DB clients, sockets, or lambdas.

### 5. Retrieval APIs

```python
cfg = {"configurable": {"thread_id": "t-1"}}

# Latest snapshot
snap = graph.get_state(cfg)
snap.values, snap.next, snap.config["configurable"]["checkpoint_id"]

# Full history (newest first), optional filters
for s in graph.get_state_history(cfg, limit=10):
    print(s.metadata["step"], s.metadata["source"], s.next,
          s.config["configurable"]["checkpoint_id"])

# A specific checkpoint
old_cfg = {"configurable": {"thread_id": "t-1",
                            "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"}}
graph.get_state(old_cfg)

# Include subgraph state
graph.get_state(cfg, subgraphs=True)

# Low level: raw tuple
tup = checkpointer.get_tuple(cfg)
tup.checkpoint["channel_values"], tup.metadata, tup.parent_config, tup.pending_writes

# Async
snap = await graph.aget_state(cfg)
async for s in graph.aget_state_history(cfg): ...
```

### 6. Retrieval flow on `invoke`

```mermaid
flowchart LR
    A["invoke(input, thread_id)"] --> B["checkpointer.get_tuple(config)"]
    B --> C{found?}
    C -->|no| D[start fresh at START]
    C -->|yes| E[deserialize channel values<br/>+ pending writes]
    E --> F[apply new input via reducers]
    F --> G[run super-steps]
    G --> H["put() / put_writes() after each step"]
```

### 7. LangGraph Platform (LangSmith Deployment)

When deployed to LangGraph Platform/Server you **do not configure a checkpointer**. The server injects a Postgres-backed one (plus a store) and exposes threads over REST/SDK:

```python
from langgraph_sdk import get_client
client = get_client(url="https://my-deployment.example.com")
thread = await client.threads.create()
await client.runs.wait(thread["thread_id"], "agent", input={"messages": [...]})
state = await client.threads.get_state(thread["thread_id"])
history = await client.threads.get_history(thread["thread_id"])
```

### 8. Comparison

| Backend | Durable | Multi-process | Async | Typical use |
|---|---|---|---|---|
| `InMemorySaver` | No | No | Yes | Tests, notebooks |
| `SqliteSaver` | Yes (file) | Limited | `AsyncSqliteSaver` | Local or desktop apps |
| `PostgresSaver` | Yes | Yes | `AsyncPostgresSaver` | Production services |
| Redis / Mongo savers | Yes | Yes | Varies | Existing infrastructure |
| Platform managed | Yes | Yes | Yes | LangGraph Platform deployments |

## Common Pitfalls

- Forgetting `checkpointer.setup()` on first use of Postgres/SQLite, which leaves the tables missing.
- Using a sync saver with `ainvoke`, or the reverse. Match sync with sync and async with async.
- Leaving the `with PostgresSaver.from_conn_string(...)` context while the graph is still in use, which closes the connection.
- Growing the DB without limit. Add retention jobs or `delete_thread`, and trim messages.
- Storing PII in plain text. Use `EncryptedSerializer` or keep sensitive data outside state.
- Mixing up `checkpoint_ns` and `thread_id` when querying subgraph state.

## Likely Follow-up Questions

1. **How do you fetch a past state?** Pass `checkpoint_id` in `configurable`, or loop over `get_state_history`.
2. **Can multiple app servers share a thread?** Yes, if they use the same Postgres checkpointer. Avoid running the same thread concurrently.
3. **How do you delete a user's data?** `checkpointer.delete_thread(thread_id)` (plus store namespaces).
4. **Why is a channel stored in blobs instead of inline?** Versioned dedup. Unchanged channels are not copied in each checkpoint.
5. **How do you migrate from SQLite to Postgres?** Swap the checkpointer. Old threads need a custom migration script that iterates `list()` and calls `put()`.

[← Back to questions](../questions.md)
