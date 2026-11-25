# ACID and Transaction Isolation Levels

## ACID

| Letter | Meaning | Mechanism |
|---|---|---|
| **A**tomicity | All statements in a transaction succeed, or none do | Undo log / rollback |
| **C**onsistency | Transaction moves DB from one valid state to another (constraints hold) | Constraints, triggers + app logic |
| **I**solation | Concurrent transactions don't see each other's partial work | Locks, MVCC |
| **D**urability | Once committed, data survives crashes | Write-ahead log (WAL), fsync |

```
BEGIN;
  UPDATE accounts SET bal = bal - 100 WHERE id = 1;
  UPDATE accounts SET bal = bal + 100 WHERE id = 2;
COMMIT;   -- both happen, or neither
```

## Concurrency Anomalies

| Anomaly | What happens |
|---|---|
| **Dirty read** | Read data another transaction has not committed yet |
| **Non-repeatable read** | Same row read twice in a transaction returns different values |
| **Phantom read** | Same query twice returns a different **set of rows** |
| **Lost update** | Two transactions read, both write; one overwrites the other |
| **Write skew** | Two transactions read overlapping data, write different rows, and together break an invariant |

Write skew example: two doctors both check "at least 2 on call", both go off call → 0 on call.

## Isolation Levels (SQL standard)

| Level | Dirty read | Non-repeatable read | Phantom |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| **Read Committed** | Prevented | Possible | Possible |
| **Repeatable Read** | Prevented | Prevented | Possible (per standard) |
| **Serializable** | Prevented | Prevented | Prevented |

Notes:
- Defaults: PostgreSQL → Read Committed; MySQL InnoDB → Repeatable Read
- Real engines differ from the standard. PostgreSQL's Repeatable Read is **snapshot isolation** (no phantoms, but write skew is still possible). Only Serializable fully prevents write skew
- Higher isolation = fewer anomalies but more contention, aborts or retries

## How Isolation Is Implemented

### MVCC (Multi-Version Concurrency Control)
- Each transaction reads a **snapshot**; writers create new row versions
- **Readers don't block writers, writers don't block readers**
- Used by PostgreSQL, MySQL InnoDB, Oracle

### Locking
| | Pessimistic | Optimistic |
|---|---|---|
| Idea | Lock first, then work | Work freely, check at commit |
| Example | `SELECT ... FOR UPDATE` | `UPDATE ... WHERE version = 5` |
| Best when | High contention | Low contention |
| Cost | Blocking, deadlocks | Retries on conflict |

```sql
-- Optimistic update with a version column
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 42 AND version = 5;      -- 0 rows updated → conflict, retry
```

## Deadlocks
Transaction A holds row 1 and wants row 2; B holds row 2 and wants row 1.
- DB detects the cycle and aborts one transaction
- Reduce them: lock rows in a **consistent order**, keep transactions short

## ACID vs BASE

| | ACID | BASE |
|---|---|---|
| Stands for | Atomic, Consistent, Isolated, Durable | **B**asically **A**vailable, **S**oft state, **E**ventually consistent |
| Priority | Correctness | Availability and scale |
| Typical | Relational DBs | Many NoSQL / distributed stores |

## Distributed Note
ACID across **multiple services or databases** is hard. Two-phase commit is blocking and fragile; most microservices use **sagas** (see `012_saga_pattern.md`).

## Interview Quick Answer
> ACID guarantees atomic, consistent, isolated and durable transactions. Isolation levels trade anomalies for concurrency: Read Committed stops dirty reads, Repeatable Read stops non-repeatable reads, Serializable stops phantoms and write skew. Databases implement this with MVCC and locks; use optimistic locking for low contention and `SELECT FOR UPDATE` for high contention.

## Rule of Thumb
Use Read Committed by default, raise to Serializable only for invariants that span multiple rows, and keep transactions short.
