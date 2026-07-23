# System Design Roadmap — Zero to Microsoft-Interview-Ready

> **Purpose of this file:** This is the master index for my system design learning journey. Every future chat should treat this as the source of truth for what I've covered, what I'm currently on, and what's next. Update the checkboxes as topics are completed.
>
> **Reference book:** *Designing Data-Intensive Applications, 2nd Ed.* (Kleppmann) — mapped below as **[DDIA]**. Chapters noted where relevant.
>
> **How to use this in future chats:** Say something like "teach me topic X from index.md" or "quiz me on Phase 2" or "I finished 1.3, mark it done and give me 1.4." I will track progress here.

## Progress Legend
- [ ] Not started
- [~] In progress
- [x] Done

---

## Phase 0 — Interview Framework (read this first, revisit constantly)
Full notes: [notes/phase-0-interview-framework.md](notes/phase-0-interview-framework.md)
- [x] 0.1 What system design interviews actually test (tradeoffs > "correct" answer)
- [x] 0.2 The standard framework: Requirements → Estimation → API design → High-level design → Deep dive → Bottlenecks/scale → Wrap-up
- [x] 0.3 Functional vs non-functional requirements (consistency, availability, latency, durability)
- [x] 0.4 Back-of-envelope estimation (QPS, storage, bandwidth math) — practice doing this fast
- [x] 0.5 How Microsoft-style interviews differ (Azure-flavored expectations, collaborative discussion style, "how would you improve this design" follow-ups)

---

## Phase 1 — Foundations (the "bottom" layer)
Full notes: [notes/phase-1-foundations.md](notes/phase-1-foundations.md)
- [x] 1.1 How computers talk: client-server model, IP, TCP vs UDP, sockets
- [x] 1.2 HTTP/HTTPS deep dive: methods, status codes, headers, TLS handshake, HTTP/2 vs HTTP/3
- [x] 1.3 DNS: how resolution works, TTLs, anycast
- [x] 1.4 Vertical vs horizontal scaling — what breaks first when you scale up vs out
- [x] 1.5 Latency numbers every engineer should know (memory vs disk vs network vs cross-region)
- [x] 1.6 Concurrency basics: processes, threads, async I/O (why this matters for servers)
- [x] 1.7 Protocol zoo: HTTP long-polling vs WebSockets vs SSE; AMQP (message brokers); SMTP/IMAP/POP3 (email); MQTT (IoT pub-sub); WebRTC (P2P real-time); RPC (remote procedure calls); FTP vs SSH (file transfer & remote access)

---

## Phase 2 — Data Models & Storage Engines **[DDIA Part I, Ch. 1–3]**
- [ ] 2.1 [DDIA Ch.1] Reliability, scalability, maintainability — the three pillars
- [ ] 2.2 [DDIA Ch.2] Data models: relational vs non-relational (document, wide-column, key-value, graph stores) — when to pick which
- [ ] 2.3 [DDIA Ch.2] Query languages: SQL vs declarative vs imperative, graph query languages
- [ ] 2.4 [DDIA Ch.3] Storage engines: hash indexes, SSTables, LSM-trees
- [ ] 2.5 [DDIA Ch.3] B-trees vs LSM-trees — the tradeoff every DB pick hinges on
- [ ] 2.6 [DDIA Ch.3] Column-oriented storage (for analytics/OLAP)
- [ ] 2.7 [DDIA Ch.4] Encoding formats: JSON/XML vs binary (Protobuf, Avro, Thrift) — schema evolution
- [ ] 2.8 Disk fragmentation: internal vs external — why storage engines compact/vacuum over time

---

## Phase 3 — Distributed Data (the hard, interview-critical part) **[DDIA Part II, Ch. 5–9]**
- [ ] 3.1 [DDIA Ch.5] Replication: leader-follower, multi-leader, leaderless (Dynamo-style)
- [ ] 3.2 [DDIA Ch.5] Sync vs async replication, replication lag, read-your-writes consistency
- [ ] 3.3 [DDIA Ch.6] Partitioning/Sharding: by key range, by hash — and the hot-spot problem
- [ ] 3.4 [DDIA Ch.6] Rebalancing partitions, request routing to the right shard
- [ ] 3.5 [DDIA Ch.7] Transactions: ACID properly explained, weak isolation pitfalls
- [ ] 3.6 [DDIA Ch.7] Isolation levels: read committed, snapshot isolation, serializable
- [ ] 3.7 [DDIA Ch.8] The trouble with distributed systems: clock drift, network faults, partial failures
- [ ] 3.8 [DDIA Ch.9] Consistency models: linearizability, causal consistency, eventual consistency
- [ ] 3.9 [DDIA Ch.9] CAP theorem — properly, not the oversimplified version
- [ ] 3.10 [DDIA Ch.9] Consensus algorithms: Paxos, Raft — what problem they actually solve
- [ ] 3.11 [DDIA Ch.9] Distributed transactions: 2PC, and why it's avoided at scale
- [ ] 3.12 Multi-leader conflict resolution: last-write-wins, timestamp-based, custom merge logic

---

## Phase 4 — Derived Data & Processing **[DDIA Part III, Ch. 10–12]**
- [ ] 4.1 [DDIA Ch.10] Batch processing: MapReduce concepts, joins at scale
- [ ] 4.2 [DDIA Ch.11] Stream processing: message brokers (Kafka-style), log-based systems
- [ ] 4.3 [DDIA Ch.11] Change data capture, event sourcing
- [ ] 4.4 [DDIA Ch.12] Combining batch + stream (lambda/kappa architecture), keeping systems correct

---

## Phase 5 — Core Building Blocks (not in DDIA — essential for interviews)
- [ ] 5.1 Load balancers: L4 vs L7; algorithms (round robin, least connections, least response time, IP hash, weighted, geographic, consistent hashing)
- [ ] 5.2 Caching: cache-aside, write-through, write-back, write-around; eviction policies (LRU/LFU); browser/app/DB-level caching; Redis/Memcached basics; cache hit ratio
- [ ] 5.3 CDN: pull-based vs push-based — how static/dynamic content gets served close to users
- [ ] 5.4 API design: REST vs GraphQL vs gRPC — when each fits; RESTful resource modeling (nouns, pagination, status codes); GraphQL schema/query/mutation basics
- [ ] 5.5 Message queues & pub/sub: Kafka vs RabbitMQ vs SQS — decoupling producers/consumers
- [ ] 5.6 Rate limiting algorithms: token bucket, leaky bucket, sliding window
- [ ] 5.7 SQL vs NoSQL decision framework (revisit 2.2 with real product scenarios)
- [ ] 5.8 Blob/object storage (S3-style): buckets, signed URLs for temporary secured access
- [ ] 5.9 Search infrastructure basics: inverted indexes, Elasticsearch at a high level
- [ ] 5.10 Notification/push systems: fan-out on write vs fan-out on read
- [ ] 5.11 Consistent hashing — the technique behind sharding, load balancers, and caches alike
- [ ] 5.12 Idempotency & deduplication in distributed calls
- [ ] 5.13 Observability: logging, metrics, tracing, health checks — how to detect failure in prod
- [ ] 5.14 API lifecycle (design → develop → deploy → monitor → retire) and design approaches (top-down, bottom-up, contract-first)
- [ ] 5.15 Application architecture patterns: MVC vs MVCS vs MVCRS (repository layer); server-driven vs client-driven UI

---

## Phase 6 — Reliability, Security & Scale at the Edges
- [ ] 6.1 Failover, redundancy, multi-region architectures; single point of failure (SPOF) identification & mitigation (health checks, self-healing)
- [ ] 6.2 Circuit breakers, retries with backoff, bulkheads (failure containment patterns)
- [ ] 6.3 Authentication methods: Basic, Digest, API keys, session-based (stateful, server-side) vs JWT (stateless: header.payload.signature)
- [ ] 6.4 Token strategy: access tokens vs refresh tokens; OAuth2.0 flows; SSO
- [ ] 6.5 Authorization models: role-based (RBAC), attribute-based (ABAC), access control lists (ACL)
- [ ] 6.6 Web/app security: CORS, XSS, CSRF, SQL/NoSQL injection, rate limiting, firewalls, VPNs, encryption in transit/at rest
- [ ] 6.7 Data partitioning for compliance (GDPR-style regional data residency)
- [ ] 6.8 Capacity planning & autoscaling strategies

---

## Phase 7 — Applied Practice: Classic Interview Problems
Work through each using the Phase 0 framework. Order is easy → hard.
- [ ] 7.1 Design a URL shortener
- [ ] 7.2 Design a rate limiter (service-level)
- [ ] 7.3 Design a key-value store (mini Dynamo)
- [ ] 7.4 Design a distributed cache
- [ ] 7.5 Design a notification system
- [ ] 7.6 Design a web crawler
- [ ] 7.7 Design an API rate-limited multi-tenant SaaS backend
- [ ] 7.8 Design Twitter/X (news feed + fan-out)
- [ ] 7.9 Design WhatsApp/Messenger (real-time chat, delivery guarantees)
- [ ] 7.10 Design Uber/ride-sharing (geo-indexing, matching)
- [ ] 7.11 Design YouTube/Netflix (video storage + streaming + CDN)
- [ ] 7.12 Design Instagram (media storage, feed ranking)
- [ ] 7.13 Design Google Drive/Dropbox (file sync, conflict resolution)
- [ ] 7.14 Design a payment system (idempotency, exactly-once semantics)
- [ ] 7.15 Design a search autocomplete/typeahead system
- [ ] 7.16 Design a distributed job scheduler
- [ ] 7.17 Design Azure-flavored variants (e.g., a Blob Storage clone, a queue service) — good for Microsoft interviews specifically

---

## Phase 8 — Interview Polish
- [ ] 8.1 Practice explaining tradeoffs out loud, not just naming technologies
- [ ] 8.2 Mock interview simulations (I'll ask you a prompt cold, you drive the whiteboard)
- [ ] 8.3 Common red flags interviewers watch for (jumping to solution, ignoring requirements, no numbers)
- [ ] 8.4 Behavioral + system design combo prep specific to Microsoft loops

---

## Notes
- Current focus: Phase 2 — Data Models & Storage Engines
- Last updated: 2026-07-22
