# Day 1 — System Design Fundamentals

**15-day distributed-systems design preparation · Notes-first format**  
**Focus:** Temporal-style senior software engineering design interviews  
**Study time:** ~30 minutes reading, followed by a short quiz

## 1. What interviewers evaluate

The supplied *Temporal Distributed Systems Design Prep Guide* describes a roughly 60-minute interview. Candidates clarify requirements, establish a baseline design, explain choices and tradeoffs, and respond to progressively harder failure or scaling constraints. The guide says questions are generally distributed-systems design rather than Temporal-specific.

Four recurring dimensions:

| Dimension | Core question |
|---|---|
| Scalability | How does throughput and storage capacity grow? |
| Reliability | What happens when processes, machines, or dependencies fail? |
| Consistency | What does a successful write guarantee subsequent reads will see? |
| Maintainability | Can engineers operate, debug, evolve, and monitor this system? |

**Interview principle:** Do not simply list Redis, Kafka, replication, and sharding. Explain what each component does, why it is needed, and how requests and failures move through it.

## 2. Requirements before architecture

**Example prompt:** Design a URL shortener.

**Functional requirements:**
- Create a short URL for a supplied long URL.
- Redirect visitors to the original URL.
- Optionally support custom aliases and expiration.

**Nonfunctional requirements (illustrative targets, not requirements from Temporal):**
- Redirect p99 latency below 50 ms.
- Service tolerates an application-server failure.
- 100 million new mappings per day.
- Newly created URLs should be readable immediately.
- Acknowledged mappings should survive defined failures.

Clarify requirements that change design choices: Is read-after-write required? Are mappings mutable? How long are they retained? What is the peak read/write rate? What happens when the database is unavailable?

## 3. Capacity estimation: worked example

Assumptions:
- 100 million new mappings/day.
- 500 bytes/mapping.
- 365-day retention.
- 100,000 peak redirect requests/sec.

Calculations (decimal units):

```text
Average writes/sec = 100,000,000 / 86,400 ≈ 1,157
Storage/day        = 100,000,000 × 500 bytes = 50 GB
Storage/year       = 50 GB × 365 = 18.25 TB
Three full copies  = 18.25 TB × 3 = 54.75 TB
```

These numbers omit indexes, metadata, backups, and operational headroom. Do not mix average writes and peak reads as if they were simultaneous steady-state rates; estimate peak writes separately.

**Why estimate?** Estimates tell you whether a single database is plausible, whether storage partitioning may be necessary, and whether the system is read-heavy or write-heavy.

## 4. Baseline architecture and request flow

```text
              Clients
                 |
           Load balancer
              /     \
        App server  App server
              \     /
               Database
```

A redirect request:
1. Client sends `GET /abc123`.
2. Load balancer selects an app server.
3. App server reads the mapping from persistent storage.
4. Server returns an HTTP redirect to the destination.

App servers can be stateless; database capacity and availability still require explicit attention. At high scale, identify and address these constraints from the outset.

## 5. Scaling mechanisms

Suppose the service receives 100,000 requests/sec, but the database supports 20,000 operations/sec. Adding application servers does **not** remove the database bottleneck.

| Mechanism | Useful when | Main tradeoff |
|---|---|---|
| Cache (e.g., Redis) | Many repeated reads | Misses, invalidation, memory cost, staleness |
| Read replicas | Reads dominate and lag is acceptable or managed | Replica lag, read routing, operational cost |
| Sharding | Data volume or write load exceeds a node's capacity | Partition routing, hot spots, rebalancing |

**Cache calculation:** If reads = 95,000/sec, writes = 5,000/sec, and cache hit rate = 90%, database operations ≈ 9,500 read misses + 5,000 writes = **14,500/sec**. This is below the assumed 20,000/sec limit, before accounting for headroom and operation costs.

A cache with a low hit rate may not help. Avoid adding components without showing what bottleneck they solve.

## 6. Replication and read-after-write consistency

```text
Client                 Primary                  Read replica
  | -- write abc123 --> |                            |
  | <-- success ------- |                            |
  | -------- read abc123 ---------------------------> |
  | <------- not found ------------------------------ |
  |                     | -- replicate abc123 -----> |
```

With **asynchronous replication**, the primary can acknowledge a write before a replica receives it. An immediate read from that replica can return stale data.

Options:
- Route freshness-sensitive reads to the primary.
- Route reads to replicas known to have reached the required replication version/position.
- Use stronger coordination where the requirements justify its latency and availability costs.

**Distinguish the concepts:**
- Replication: multiple copies of data.
- Sharding: dividing data across partitions.
- Consistency: guarantees about observable values and operation ordering.

Waiting for **one** replica to acknowledge does not guarantee an immediate read from **any** replica is fresh.

## 7. Failure windows and idempotency

Create operation:

```text
1. Receive POST /shorten
2. Generate short code
3. Commit mapping to database
4. Return success to client
```

If the server crashes after step 3 but before step 4, the mapping exists but the client does not know the result. Retrying may accidentally create another mapping.

One approach is a durable **idempotency key**: the client sends a stable request identifier, and the server atomically associates that identifier with the operation result. A retry can retrieve the prior result rather than repeating the logical creation.

Always ask: **What was committed? What did the client observe? What will a retry do?**

## 8. Minimal API and schema

```http
POST /shorten
Idempotency-Key: request-123
Content-Type: application/json

{"longUrl":"https://example.com/article"}
```

```http
201 Created
Content-Type: application/json

{"shortCode":"abc123"}
```

```http
GET /abc123

302 Found
Location: https://example.com/article
```

```sql
CREATE TABLE url_mappings (
    short_code VARCHAR(16) PRIMARY KEY,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP NULL
);
```

A production design may also need idempotency records, uniqueness constraints, expiration policies, and monitoring.

## 9. Reusable interview sequence

1. **Clarify** — functions, guarantees, constraints.
2. **Estimate** — throughput, data volume, latency, peak demand.
3. **Design** — APIs, schema, components, request/data flow.
4. **Scale** — identify bottlenecks and justify mitigations.
5. **Break it** — simulate crashes, retries, stale reads, partial writes.
6. **Defend it** — discuss alternatives and operational tradeoffs.

---

## Self-check quiz

*Try without looking back. Answers follow below.*

**Q1.** A service creates 20 million records/day at 1 KB each. How much storage is needed for 100 days, in decimal units?

A. 200 GB  
B. 2 TB  
C. 20 TB  
D. 200 TB

**Q2.** Reads = 80,000/sec, writes = 5,000/sec, and a cache handles 90% of reads. How many operations/sec reach the database?

A. 8,000  
B. 13,000  
C. 18,000  
D. 21,000

**Q3. True or false:** With asynchronous read replicas, a successful write acknowledgment guarantees an immediate read from any replica sees the write.

**Q4. Select all that apply:** Which can help prevent duplicate creation on client retries?

A. Persist an idempotency key with the result  
B. Enforce a unique logical request identifier  
C. Add more stateless application servers  
D. Atomically persist the operation and its deduplication record

**Q5. True or false:** Adding more stateless application servers automatically increases throughput of an already-saturated database.

### Answers and explanations

1. **B — 2 TB.** 20,000,000 × 1,000 bytes × 100 = 2,000,000,000,000 bytes. A common mistake is to choose 20 TB; that is ten times too large.
2. **B — 13,000/sec.** 10% of 80,000 reads = 8,000 misses; add 5,000 writes.
3. **False.** Replication lag can produce stale reads.
4. **A, B, D.** Durable deduplication and atomic recording help ensure repeated requests don't repeat the logical operation. Extra app servers do not solve this.
5. **False.** The database remains the bottleneck.

---

## Source and scope

**Temporal-specific interview expectations:** Derived from the *Distributed Systems Design Prep Guide* supplied in this conversation (interview flow, four dimensions, and interviewer feedback).  
**URL-shortener architecture, numerical assumptions, examples, explanations, and quiz:** General teaching material constructed for this study session, not claims that Temporal prescribes this particular design.

**Next session:** Day 2 — Storage engines: WAL, LSM trees, durability, and crash recovery.
