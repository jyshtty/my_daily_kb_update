# Unique ID Generation in Distributed Systems

## Requirements
- **Unique** across all nodes, with no coordination on the hot path
- Preferably **roughly time-sortable** (good for DB indexes, pagination)
- Compact (e.g. 64-bit) and fast to generate
- Works with many generators and high throughput (e.g. 10k+/sec per node)

## Options

| Approach | Pros | Cons |
|---|---|---|
| **DB auto-increment** | Simple, ordered, small | Single point of failure / bottleneck; hard to shard; leaks business volume; guessable |
| **Multi-master auto-increment** (step = N) | No single DB | Not time-ordered globally; hard to add nodes |
| **UUID v4** | No coordination, huge space | 128-bit, random → poor index locality, not sortable |
| **UUID v7 / ULID** | Time-ordered prefix + randomness, no coordination | 128-bit; ordering is only roughly by time |
| **Ticket server** (central DB hands out ranges) | Simple, ordered | Central dependency; mitigate by fetching ID **ranges** in batches |
| **Snowflake-style** | 64-bit, time-sortable, no per-ID coordination | Needs unique machine IDs; depends on clocks |

## Snowflake (Twitter) Layout
```
 1 bit    41 bits              10 bits         12 bits
┌─────┬──────────────────┬───────────────┬──────────────┐
│  0  │ timestamp (ms)   │ machine / DC  │ sequence     │
└─────┴──────────────────┴───────────────┴──────────────┘
sign    ~69 years         1024 nodes       4096 IDs per ms per node
```
- Timestamp = ms since a custom epoch → IDs sort by creation time
- Machine ID = datacenter + worker, assigned via config or ZooKeeper/etcd
- Sequence resets each millisecond; if exhausted, wait for the next ms
- Capacity: 4096 IDs/ms/node ≈ 4 million per second per node

```python
def next_id():
    ts = now_ms()
    if ts < last_ts:               # clock moved backwards
        raise ClockMovedBackwards  # or wait until last_ts
    if ts == last_ts:
        seq = (seq + 1) & 0xFFF
        if seq == 0:
            ts = wait_next_ms(last_ts)
    else:
        seq = 0
    last_ts = ts
    return ((ts - EPOCH) << 22) | (machine_id << 12) | seq
```

## Clock Problems
| Problem | Mitigation |
|---|---|
| Clock skew between nodes | NTP, tolerate small skew; ordering is only approximate across nodes |
| Clock moves backwards (NTP step, leap second) | Refuse to generate or wait; use monotonic checks |
| Duplicate machine IDs | Central registry / lease for worker IDs |

## Choosing

| Need | Pick |
|---|---|
| Quick, small system, one DB | Auto-increment |
| No infra, globally unique, ordering not critical | UUID v4 |
| Unique **and** index-friendly sortable keys, no extra service | UUID v7 / ULID |
| 64-bit sortable IDs at high scale (URL shorteners, tweets, orders) | Snowflake |

## Related Notes
- **URL shortener**: a counter/Snowflake ID encoded in base62 gives short, unique codes
- **Sharding**: embedding a shard ID in the ID lets you route without a lookup (see `database/001_sharding.md`)
- **Security**: sequential IDs are guessable; don't expose them where enumeration is a risk

## Interview Quick Answer
> For distributed unique IDs I'd use a Snowflake-style 64-bit ID: timestamp, machine ID and sequence. It needs no per-ID coordination, is time-sortable and index-friendly. Risks are clock skew and clock rollback, handled with NTP and by refusing to issue IDs when the clock goes backwards. If 128 bits is fine, UUID v7 or ULID gives the same benefits with no machine-ID management.

## Rule of Thumb
Use UUID v7/ULID when you want zero coordination, Snowflake when you need compact 64-bit sortable IDs, and avoid random UUID v4 as a clustered primary key at scale.
