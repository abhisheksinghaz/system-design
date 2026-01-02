# Complete Architecture of Distributed Key-Value Store

## System Components

```
┌─────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                         │
│                     (Load Balancer / API)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      COORDINATOR NODE                        │
│              (Any node handling the request)                 │
│  • Receives get(key) / put(key, value)                      │
│  • Determines replica nodes using consistent hashing         │
│  • Coordinates quorum reads/writes (R/W replicas)           │
│  • Handles conflict resolution (vector clocks)              │
└──────────────┬──────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                    CONSISTENT HASH RING                      │
│                                                              │
│     Node A ──── Node B ──── Node C ──── Node D ──── Node A  │
│        │          │            │           │          │     │
│     (0-90)    (90-180)    (180-270)   (270-360)   (wrap)   │
│                                                              │
│  • Virtual nodes for better distribution                     │
│  • Gossip protocol for membership & failure detection       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    STORAGE NODES (each node)                 │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              STORAGE ENGINE (LSM Tree)                  │ │
│  │                                                         │ │
│  │  ┌─────────────┐                                       │ │
│  │  │  MemTable   │  (In-memory write buffer)             │ │
│  │  │  Red-Black  │                                       │ │
│  │  │    Tree     │                                       │ │
│  │  └──────┬──────┘                                       │ │
│  │         │ (flush when full)                            │ │
│  │         ▼                                               │ │
│  │  ┌─────────────┐                                       │ │
│  │  │  WAL (Log)  │  (Write-Ahead Log for durability)    │ │
│  │  └─────────────┘                                       │ │
│  │         │                                               │ │
│  │         ▼                                               │ │
│  │  ┌─────────────────────────────────────┐              │ │
│  │  │         SSTables (on disk)          │              │ │
│  │  │  ┌──────────┐  ┌──────────┐        │              │ │
│  │  │  │SSTable-1 │  │SSTable-2 │  ...   │              │ │
│  │  │  │(sorted)  │  │(sorted)  │        │              │ │
│  │  │  └──────────┘  └──────────┘        │              │ │
│  │  │                                     │              │ │
│  │  │  • Immutable files                 │              │ │
│  │  │  • Compaction merges old tables    │              │ │
│  │  └─────────────────────────────────────┘              │ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────┐              │ │
│  │  │      Bloom Filters (in memory)      │              │ │
│  │  │  • One per SSTable                  │              │ │
│  │  │  • Quick "key not present" check    │              │ │
│  │  └─────────────────────────────────────┘              │ │
│  │                                                         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │            METADATA & COORDINATION                      │ │
│  │  • Vector Clocks (for versioning)                      │ │
│  │  • Merkle Trees (for anti-entropy / sync)             │ │
│  │  • Hinted Handoff (temporary storage for down nodes)  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Detailed Flow: PUT Operation

### **put(key="user:123", value="John Doe")**

```
Step 1: Client Request
┌────────┐
│ Client │ ──── put(key, value) ───▶ Load Balancer ───▶ Node B (Coordinator)
└────────┘

Step 2: Determine Replicas
Node B (Coordinator):
  • hash(key) = hash("user:123") = 215
  • Lookup ring: position 215 → Node C (primary)
  • Replication factor N=3 → replicas: Node C, Node D, Node A

Step 3: Parallel Writes to Replicas
Node B ──┬──▶ Node C: write(key, value, vector_clock)
         ├──▶ Node D: write(key, value, vector_clock)
         └──▶ Node A: write(key, value, vector_clock)

Step 4: Each Replica Node (e.g., Node C):
  ┌─────────────────────────────────────┐
  │ 1. Append to WAL (for durability)   │
  │    WAL: [PUT, user:123, John Doe]   │
  │                                     │
  │ 2. Write to MemTable                │
  │    MemTable[user:123] = {           │
  │      value: "John Doe",             │
  │      vector_clock: [C:5, D:3, A:2]  │
  │    }                                │
  │                                     │
  │ 3. Increment vector clock           │
  │    [C:5, D:3, A:2] → [C:6, D:3, A:2]│
  │                                     │
  │ 4. Send ACK to coordinator          │
  └─────────────────────────────────────┘

Step 5: Quorum Check
Node B (Coordinator):
  • W = 2 (write quorum)
  • Receives ACKs from Node C and Node D
  • W=2 satisfied ✓
  • Return SUCCESS to client

  (Node A might still be writing - async replication)

Step 6: Background Operations
  • When MemTable full → flush to SSTable on disk
  • Create Bloom Filter for new SSTable
  • Periodic compaction: merge old SSTables
```

---

## Detailed Flow: GET Operation

### **get(key="user:123")**

```
Step 1: Client Request
┌────────┐
│ Client │ ──── get(key) ───▶ Load Balancer ───▶ Node A (Coordinator)
└────────┘

Step 2: Determine Replicas
Node A (Coordinator):
  • hash("user:123") = 215
  • Lookup ring → replicas: Node C, Node D, Node A

Step 3: Parallel Reads from R Replicas (R=2)
Node A ──┬──▶ Node C: read(key)
         └──▶ Node D: read(key)

Step 4: Each Replica Node (e.g., Node C) Read Process:
  ┌─────────────────────────────────────────┐
  │ 1. Check MemTable first                 │
  │    IF found → return immediately        │
  │                                         │
  │ 2. Check Bloom Filters (for SSTables)  │
  │    • SSTable-3 Bloom: "might exist"    │
  │    • SSTable-2 Bloom: "NOT present"    │
  │    • SSTable-1 Bloom: "might exist"    │
  │                                         │
  │ 3. Search SSTables (newest → oldest)   │
  │    • Skip SSTable-2 (Bloom says NO)    │
  │    • Search SSTable-3: FOUND!          │
  │      user:123 = {                      │
  │        value: "John Doe",              │
  │        vector_clock: [C:6, D:3, A:2]   │
  │      }                                 │
  │                                         │
  │ 4. Return value + vector clock         │
  └─────────────────────────────────────────┘

Step 5: Coordinator Receives Responses
Node A receives:
  • From Node C: {value: "John Doe", VC: [C:6, D:3, A:2]}
  • From Node D: {value: "John Doe", VC: [C:6, D:3, A:2]}

Step 6: Conflict Detection (Vector Clock Comparison)
  ┌────────────────────────────────────────────┐
  │ Compare vector clocks:                     │
  │                                            │
  │ Case 1: VC1 dominates VC2                 │
  │   [C:6, D:3, A:2] vs [C:5, D:3, A:2]     │
  │   → No conflict, return latest (VC1)      │
  │                                            │
  │ Case 2: Concurrent (CONFLICT!)            │
  │   [C:6, D:3, A:2] vs [C:5, D:4, A:2]     │
  │   → Return BOTH versions to client        │
  │   → Client must resolve conflict          │
  └────────────────────────────────────────────┘

Step 7: Return to Client
  • If no conflict: return single value
  • If conflict: return all versions + vector clocks
  • Client resolves (e.g., merge shopping carts, last-write-wins)
```

---

## Additional Architectural Features

### **1. Failure Handling**

**Node Failure During Write:**
```
• W=2, but only 1 replica responds (Node C down)
• Coordinator uses "Hinted Handoff":
  ┌─────────────────────────────────┐
  │ Node A (Coordinator):           │
  │   • Stores hint locally:        │
  │     "user:123 → Node C (down)"  │
  │   • When Node C recovers:       │
  │     → replay hint to Node C     │
  └─────────────────────────────────┘
```

**Node Failure During Read:**
```
• R=2, but only 1 replica responds
• Coordinator reads from additional replica
• OR returns with lower consistency
```

### **2. Anti-Entropy (Background Sync)**

```
Merkle Trees (per node):
┌─────────────────────────────┐
│      Root Hash              │
│         /    \              │
│   Hash-L    Hash-R          │
│    /  \      /  \           │
│  H-1  H-2  H-3  H-4         │
│  (key ranges)               │
└─────────────────────────────┘

• Nodes periodically exchange Merkle tree roots
• Detect differences quickly
• Sync only divergent key ranges
```

### **3. Gossip Protocol**

```
Every T seconds, each node:
1. Picks random neighbor
2. Exchanges membership info
   • Node list
   • Health status (heartbeats)
3. Eventually consistent cluster view

Node A: "I see: A(alive), B(alive), C(down), D(alive)"
Node B: "I see: A(alive), B(alive), C(alive), D(alive)"
→ Node A updates: C is actually alive
```

---

## Configuration Parameters

```
Tunable Consistency (CAP trade-off):

N = 3  (replication factor)
R = 2  (read quorum)
W = 2  (write quorum)

• R + W > N  →  Strong consistency
• R + W ≤ N  →  Eventual consistency

Examples:
• R=1, W=1  →  Maximum availability, weakest consistency
• R=2, W=2  →  Balanced (common default)
• R=3, W=3  →  Strongest consistency, lower availability
```

---

## Key Architectural Decisions

| Component | Why This Design? |
|-----------|-----------------|
| **Consistent Hashing** | Even data distribution, minimal rehashing when nodes join/leave |
| **Vector Clocks** | Detect concurrent writes, track causality |
| **LSM Trees (SSTables)** | Optimize for write-heavy workloads (sequential writes) |
| **Bloom Filters** | Avoid expensive disk reads for non-existent keys |
| **Gossip Protocol** | Decentralized failure detection, no single point of failure |
| **Hinted Handoff** | Temporary writes when replica unavailable |
| **Merkle Trees** | Efficient synchronization, detect divergence quickly |
| **Any Node as Coordinator** | No bottleneck, symmetric architecture |

---

## Summary

This architecture is essentially **Amazon Dynamo** / **Apache Cassandra** design. It prioritizes availability and partition tolerance (AP in CAP), with tunable consistency via quorum parameters.

**Key characteristics:**
- Decentralized (no master/slave)
- Eventually consistent (tunable)
- Highly available
- Horizontally scalable
- Optimized for high write throughput