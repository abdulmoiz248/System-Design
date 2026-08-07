# System Design Roadmap — Zero to Microsoft-Interview-Ready

> **Purpose of this file:** This is the master index for my system design learning journey. Every future chat should treat this as the source of truth for what I've covered, what I'm currently on, and what's next. Update the checkboxes as topics are completed.
>
> **Reference book:** *Designing Data-Intensive Applications, 2nd Ed.* (Kleppmann) — mapped below as **[DDIA]**. Chapters noted where relevant.
>
> **How to use this in future chats:** Say something like "teach me topic X from index.md" or "quiz me on Phase 2" or "I finished 1.3, mark it done and give me 1.4." I will track progress here.
>
> **Notes convention:** each phase gets exactly **one** consolidated file in `notes/` (e.g. `notes/phase-1-foundations.md`) — never split a phase across multiple files. Every subtopic in that file must be explained in real depth (mechanics, internals, concrete tools/examples) — no one-line "X is used for Y" summaries.

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
Full notes: [notes/phase-2-data-models-storage.md](notes/phase-2-data-models-storage.md)
- [x] 2.1 [DDIA Ch.1] Reliability, scalability, maintainability — the three pillars
- [x] 2.2 [DDIA Ch.2] Data models: relational vs non-relational (document, wide-column, key-value, graph stores) — when to pick which
- [x] 2.3 [DDIA Ch.2] Query languages: SQL vs declarative vs imperative, graph query languages
- [x] 2.4 [DDIA Ch.3] Storage engines: hash indexes, SSTables, LSM-trees
- [x] 2.5 [DDIA Ch.3] B-trees vs LSM-trees — the tradeoff every DB pick hinges on
- [x] 2.6 [DDIA Ch.3] Column-oriented storage (for analytics/OLAP)
- [x] 2.7 [DDIA Ch.4] Encoding formats: JSON/XML vs binary (Protobuf, Avro, Thrift) — schema evolution
- [x] 2.8 Disk fragmentation: internal vs external — why storage engines compact/vacuum over time

---

## Phase 3 — Distributed Data (the hard, interview-critical part) **[DDIA Part II, Ch. 5–9]**
Full notes: [notes/phase-3-distributed-data.md](notes/phase-3-distributed-data.md)
- [x] 3.1 [DDIA Ch.5] Replication: leader-follower, multi-leader, leaderless (Dynamo-style)
- [x] 3.2 [DDIA Ch.5] Sync vs async replication, replication lag, read-your-writes consistency
- [x] 3.3 [DDIA Ch.6] Partitioning/Sharding: by key range, by hash — and the hot-spot problem
- [x] 3.4 [DDIA Ch.6] Rebalancing partitions, request routing to the right shard
- [x] 3.5 [DDIA Ch.7] Transactions: ACID properly explained, weak isolation pitfalls
- [x] 3.6 [DDIA Ch.7] Isolation levels: read committed, snapshot isolation, serializable
- [x] 3.7 [DDIA Ch.8] The trouble with distributed systems: clock drift, network faults, partial failures
- [x] 3.8 [DDIA Ch.9] Consistency models: linearizability, causal consistency, eventual consistency
- [x] 3.9 [DDIA Ch.9] CAP theorem — properly, not the oversimplified version
- [x] 3.10 [DDIA Ch.9] Consensus algorithms: Paxos, Raft — what problem they actually solve
- [x] 3.11 [DDIA Ch.9] Distributed transactions: 2PC, and why it's avoided at scale
- [x] 3.12 Multi-leader conflict resolution: last-write-wins, timestamp-based, custom merge logic

---

## Phase 4 — Derived Data & Processing **[DDIA Part III, Ch. 10–12]**
Full notes: [notes/phase-4-derived-data-processing.md](notes/phase-4-derived-data-processing.md)
- [x] 4.1 [DDIA Ch.10] Batch processing: MapReduce concepts, joins at scale
- [x] 4.2 [DDIA Ch.11] Stream processing: message brokers (Kafka-style), log-based systems
- [x] 4.3 [DDIA Ch.11] Change data capture, event sourcing
- [x] 4.4 [DDIA Ch.12] Combining batch + stream (lambda/kappa architecture), keeping systems correct

---

## Phase 5 — Core Building Blocks (not in DDIA — essential for interviews)
Full notes: [notes/phase-5-core-building-blocks.md](notes/phase-5-core-building-blocks.md)
- [x] 5.1 Load balancers: L4 vs L7; algorithms (round robin, least connections, least response time, IP hash, weighted, geographic, consistent hashing)
- [x] 5.2 Caching: cache-aside, write-through, write-back, write-around; eviction policies (LRU/LFU); browser/app/DB-level caching; Redis/Memcached basics; cache hit ratio
- [x] 5.3 CDN: pull-based vs push-based — how static/dynamic content gets served close to users
- [x] 5.4 API design: REST vs GraphQL vs gRPC — when each fits; RESTful resource modeling (nouns, pagination, status codes); GraphQL schema/query/mutation basics
- [x] 5.5 Message queues & pub/sub: Kafka vs RabbitMQ vs SQS — decoupling producers/consumers
- [x] 5.6 Rate limiting algorithms: token bucket, leaky bucket, sliding window
- [x] 5.7 SQL vs NoSQL decision framework (revisit 2.2 with real product scenarios)
- [x] 5.8 Blob/object storage (S3-style): buckets, signed URLs for temporary secured access
- [x] 5.9 Search infrastructure basics: inverted indexes, Elasticsearch at a high level
- [x] 5.10 Notification/push systems: fan-out on write vs fan-out on read
- [x] 5.11 Consistent hashing — the technique behind sharding, load balancers, and caches alike
- [x] 5.12 Idempotency & deduplication in distributed calls
- [x] 5.13 Observability: logging, metrics, tracing, health checks — how to detect failure in prod
- [x] 5.14 API lifecycle (design → develop → deploy → monitor → retire) and design approaches (top-down, bottom-up, contract-first)
- [x] 5.15 Application architecture patterns: MVC vs MVCS vs MVCRS (repository layer); server-driven vs client-driven UI

---

## Phase 6 — Reliability, Security & Scale at the Edges
Full notes: [notes/phase-6-reliability-security-scale.md](notes/phase-6-reliability-security-scale.md)
- [x] 6.1 Failover, redundancy, multi-region architectures; single point of failure (SPOF) identification & mitigation (health checks, self-healing)
- [x] 6.2 Circuit breakers, retries with backoff, bulkheads (failure containment patterns)
- [x] 6.3 Authentication methods: Basic, Digest, API keys, session-based (stateful, server-side) vs JWT (stateless: header.payload.signature)
- [x] 6.4 Token strategy: access tokens vs refresh tokens; OAuth2.0 flows; SSO
- [x] 6.5 Authorization models: role-based (RBAC), attribute-based (ABAC), access control lists (ACL)
- [x] 6.6 Web/app security: CORS, XSS, CSRF, SQL/NoSQL injection, rate limiting, firewalls, VPNs, encryption in transit/at rest
- [x] 6.7 Data partitioning for compliance (GDPR-style regional data residency)
- [x] 6.8 Capacity planning & autoscaling strategies

---

## Phase 7 — Applied Practice: Classic Interview Problems
Full notes: [notes/phase-7-applied-practice.md](notes/phase-7-applied-practice.md)
Work through each using the Phase 0 framework. Order is easy → hard.
- [x] 7.1 Design a URL shortener
- [x] 7.2 Design a rate limiter (service-level)
- [x] 7.3 Design a key-value store (mini Dynamo)
- [x] 7.4 Design a distributed cache
- [x] 7.5 Design a notification system
- [x] 7.6 Design a web crawler
- [x] 7.7 Design an API rate-limited multi-tenant SaaS backend
- [x] 7.8 Design Twitter/X (news feed + fan-out)
- [x] 7.9 Design WhatsApp/Messenger (real-time chat, delivery guarantees)
- [x] 7.10 Design Uber/ride-sharing (geo-indexing, matching)
- [x] 7.11 Design YouTube/Netflix (video storage + streaming + CDN)
- [x] 7.12 Design Instagram (media storage, feed ranking)
- [x] 7.13 Design Google Drive/Dropbox (file sync, conflict resolution)
- [x] 7.14 Design a payment system (idempotency, exactly-once semantics)
- [x] 7.15 Design a search autocomplete/typeahead system
- [x] 7.16 Design a distributed job scheduler
- [x] 7.17 Design Azure-flavored variants (e.g., a Blob Storage clone, a queue service) — good for Microsoft interviews specifically

---

## Phase 8 — Interview Polish
Full notes: [notes/phase-8-interview-polish.md](notes/phase-8-interview-polish.md)
- [x] 8.1 Practice explaining tradeoffs out loud, not just naming technologies
- [x] 8.2 Mock interview simulations (I'll ask you a prompt cold, you drive the whiteboard)
- [x] 8.3 Common red flags interviewers watch for (jumping to solution, ignoring requirements, no numbers)
- [x] 8.4 Behavioral + system design combo prep specific to Microsoft loops

---

## Phase 9 — AI/LLM System Design (not in DDIA — increasingly common in 2026 interviews)
Full notes: [notes/phase-9-ai-llm-systems.md](notes/phase-9-ai-llm-systems.md)
- [x] 9.1 LLM inference basics: tokens, context window, autoregressive generation, latency metrics (time-to-first-token vs time-per-output-token)
- [x] 9.2 Serving architecture: dynamic/continuous batching, KV cache, GPU memory management, tensor vs pipeline parallelism for model sharding
- [x] 9.3 RAG (Retrieval-Augmented Generation): architecture, chunking strategies, embedding generation, vector databases (Pinecone, Weaviate, pgvector, FAISS), hybrid dense+sparse search
- [x] 9.4 Prompt engineering vs fine-tuning vs RAG — decision framework for when each fits
- [x] 9.5 Semantic caching & prompt caching for cost/latency reduction
- [x] 9.6 Agent architectures: tool/function calling, multi-step reasoning (ReAct pattern), orchestration frameworks
- [x] 9.7 Guardrails & safety: content moderation, prompt injection defense, output validation, per-user cost/rate limiting
- [x] 9.8 Cost optimization: model routing (small vs large model by query complexity), streaming responses, token budgeting
- [x] 9.9 Evaluation & observability for LLM systems: eval harnesses, hallucination detection, human-in-the-loop feedback loops
- [x] 9.10 Multi-modal & real-time considerations: image/audio pipelines, streaming chat UIs (ties back to Phase 1 WebSockets/SSE)
- [x] 9.11 Scaling challenges specific to LLM systems: GPU capacity planning, queueing/load shedding under GPU scarcity, multi-region model deployment

---

## Phase 10 — Scenario Question Bank (capstone self-test, basic → advanced, incl. AI systems)
Full notes: [notes/phase-10-scenario-question-bank.md](notes/phase-10-scenario-question-bank.md)
60 rapid-fire scenario questions with answers, spanning every phase above, for final review/self-testing before interviews.
- [x] 10.1 Basic scenarios (Q1–15): fundamentals — protocols, scaling choices, caching, simple estimation
- [x] 10.2 Intermediate scenarios (Q16–35): building-block combination — rate limiting, DB choice, replication, API design calls
- [x] 10.3 Advanced scenarios (Q36–50): distributed systems tradeoffs — consistency, partition handling, failure scenarios, consensus
- [x] 10.4 AI/LLM system scenarios (Q51–60): RAG design, inference scaling, cost tradeoffs, safety/guardrails

---

## Notes
- Current focus: **Curriculum complete (Phases 0–10).** Next step is user-driven: work through Phase 7 case studies and Phase 10 scenario bank as active review, or request mock interviews per Phase 8.2.
- Last updated: 2026-08-05
