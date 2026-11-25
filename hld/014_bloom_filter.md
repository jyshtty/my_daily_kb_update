# Bloom Filter

## What It Is
A **space-efficient probabilistic set**. It answers: *"Is this item in the set?"*

| Answer | Meaning |
|---|---|
| **No** | Definitely not in the set (100% certain) |
| **Yes** | **Probably** in the set (small chance of a false positive) |

> No false negatives. Some false positives.

## How It Works
A bit array of `m` bits, all 0, and `k` independent hash functions.

```
Add "apple":   h1→2, h2→5, h3→9   → set bits 2, 5, 9 to 1
Add "bat":     h1→5, h2→7, h3→12  → set bits 5, 7, 12 to 1

Bits: 0 0 1 0 0 1 0 1 0 1 0 0 1 0 0 0
          ^     ^   ^   ^     ^

Query "cat":   h → 2, 7, 11  → bit 11 is 0 → DEFINITELY NOT present
Query "dog":   h → 2, 5, 12  → all 1s      → PROBABLY present
                                              (could be a false positive from other items)
```

- **Add**: set the `k` bits to 1
- **Query**: all `k` bits are 1 → "maybe"; any bit is 0 → "no"

## Properties

| Property | Value |
|---|---|
| Memory | ~10 bits per element for 1% false positives |
| Lookup / insert | O(k), constant time |
| Deletion | **Not supported** (clearing a bit may remove other items) |
| Resize | Can't grow in place; must rebuild or use a scalable variant |
| False positives | Yes, tunable |
| False negatives | Never |

## Sizing
For `n` expected items and target false-positive rate `p`:

```
m = -(n × ln p) / (ln 2)²        bits needed
k = (m / n) × ln 2               optimal hash count
```

| False-positive rate | Bits per item | Hashes (k) |
|---|---|---|
| 1% | ~9.6 | 7 |
| 0.1% | ~14.4 | 10 |
| 0.01% | ~19.2 | 13 |

Example: 100 million items at 1% ≈ 120 MB, versus several GB to store the items themselves.
If you insert more than `n` items, the false-positive rate rises.

## Variants
| Variant | Adds |
|---|---|
| **Counting Bloom filter** | Counters instead of bits → supports delete (more memory) |
| **Scalable Bloom filter** | Grows by chaining filters |
| **Cuckoo filter** | Supports delete, often better space at low FP rates |
| **Quotient filter** | Cache-friendly, supports delete and merge |

## Use Cases

| Use case | How it helps |
|---|---|
| **Cache penetration protection** | Check filter before cache/DB; requests for non-existent keys are rejected cheaply |
| **LSM-tree databases** (Cassandra, HBase, RocksDB, LevelDB) | Skip SSTables that definitely don't contain the key → avoid disk reads |
| **Web crawlers** | "Have we already visited this URL?" |
| **Duplicate detection / dedupe** | Quickly filter events already seen |
| **Username / email availability** | Fast "definitely available" answer |
| **Malicious URL / spam check** | Local filter first, confirm remote only on "maybe" |
| **Recommendation feeds** | Skip items already shown to a user |

```
Request(key) → Bloom filter says NO  → return "not found" immediately
                       │
                    says MAYBE → check cache → check DB
```

## Pitfalls
- A "yes" must always be **verified** against the real store when correctness matters
- Can't remove items — stale entries remain until rebuild
- Wrong sizing (underestimating `n`) degrades accuracy
- Needs good, independent, fast hashes (e.g. MurmurHash, xxHash, double hashing)

## Interview Quick Answer
> A Bloom filter is a bit array plus k hash functions that tests set membership using very little memory. It can return false positives but never false negatives, so a "no" is certain and a "yes" means "maybe". It's used to avoid expensive lookups — in LSM-tree databases to skip SSTables, to stop cache-penetration on missing keys, and in crawlers for visited-URL checks. Deletion isn't supported unless you use a counting or cuckoo variant.

## Rule of Thumb
Use a Bloom filter in front of an expensive lookup when most queries are for items that don't exist, and always confirm positives against the source of truth.
