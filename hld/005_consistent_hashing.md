# Consistent Hashing

## The Problem with Simple Modulo Hashing

With N servers: `server = hash(key) % N`

When you add or remove a server, N changes — almost **every key remaps** to a different server.
In a cache cluster this causes a thundering herd (all cache misses at once).

```
3 servers: key "user123" → hash % 3 = 1 → Server 1
4 servers: key "user123" → hash % 4 = 3 → Server 3  ← different!
```

## How Consistent Hashing Works

1. Place servers on a **virtual ring** (0 to 2^32)
2. Hash each server to a position on the ring
3. For each key: hash it → walk clockwise → first server you hit owns it
4. Add/remove server → only keys between it and its predecessor remap

```
          0
         /  \
  Server C   Server A
        |     |
      Server B
         \  /
         2^32
```

Adding Server D between C and A → only keys previously going to A (that now fall in D's range) move.
Everything else stays put.

## Virtual Nodes

Real implementations add multiple positions per server ("virtual nodes") for even distribution.

```python
import hashlib
from bisect import bisect_right, insort

class ConsistentHashRing:
    def __init__(self, vnodes: int = 150):
        self.vnodes = vnodes
        self.ring: list[int] = []
        self.map: dict[int, str] = {}

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_server(self, server: str):
        for i in range(self.vnodes):
            h = self._hash(f"{server}:{i}")
            insort(self.ring, h)
            self.map[h] = server

    def remove_server(self, server: str):
        for i in range(self.vnodes):
            h = self._hash(f"{server}:{i}")
            self.ring.remove(h)
            del self.map[h]

    def get_server(self, key: str) -> str:
        if not self.ring:
            return None
        h = self._hash(key)
        idx = bisect_right(self.ring, h) % len(self.ring)
        return self.map[self.ring[idx]]

# Usage
ring = ConsistentHashRing()
ring.add_server("cache-1")
ring.add_server("cache-2")
ring.add_server("cache-3")

server = ring.get_server("user:123")  # always same server for same key
```

## Impact of Adding/Removing a Node

| Hashing | Keys remapped when node added/removed |
|---|---|
| Modulo (hash % N) | ~100% of keys |
| Consistent hashing | ~1/N of keys (only affected range) |

With 10 servers: adding one server remaps ~10% of keys, not 100%.

## Real-World Uses

| System | Use |
|---|---|
| **Cassandra** | Partitions data across nodes |
| **DynamoDB** | Routes requests to storage nodes |
| **Redis Cluster** | Assigns hash slots to nodes |
| **CDN** | Routes requests to edge servers |
| **Load balancers** | Session affinity (sticky sessions) |

## Key Properties
- **Monotonicity**: adding nodes only moves keys from existing nodes to new node
- **Spread**: same key always maps to same server (within stable topology)
- **Balance**: virtual nodes ensure even key distribution

## Key Rule
Use consistent hashing whenever you need to distribute keys across a dynamic set of nodes
and cannot afford to remap all keys on topology changes.
