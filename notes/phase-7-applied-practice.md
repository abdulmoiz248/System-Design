# Phase 7 — Applied Practice: Classic Interview Problems

Everything before this phase was building vocabulary and mechanics in isolation — a replication strategy here, a caching pattern there. This phase is where it gets assembled under interview conditions. Every case study below follows the **Phase 0 framework** exactly (Requirements → Estimation → API design → High-level design → Deep dive → Bottlenecks & scaling) and leans hard on concrete numbers and named technology choices, because "I'd use a database" is not an answer an interviewer accepts — which database, storing what, accessed how, is the actual content of the answer. Cross-references like "see Phase 3.3" or "the fan-out pattern from 5.10" point at the roadmap sections in `index.md` that go deeper on that specific mechanic; this file is about *applying* those mechanics under a specific problem's constraints, not re-deriving them from scratch each time.

A note on scope: these are interview-length answers — enough depth to drive a 45-minute conversation and survive follow-up questions, not a production RFC. Real systems (actual Twitter, actual S3) differ from these write-ups in a thousand operational details; that's expected and fine.

---

## 7.1 Design a URL shortener

**Requirements**
Functional: given a long URL, return a short alias; visiting the short alias 302-redirects to the original URL; optionally support custom aliases and expiration dates. Non-functional: the system is overwhelmingly **read-heavy** (every redirect is a read, but a URL is only shortened once), redirects must be low-latency (users notice a slow link), links must not collide, and short codes should be hard to sequentially guess/enumerate. Out of scope for this pass: click analytics, user accounts.

**Estimation**
Assume 100M new URLs created per month → ~40 writes/sec average. Assume a 100:1 read:write ratio (typical for link shorteners) → ~4,000 reads/sec average, with realistic peak traffic (a viral link) spiking 5-10x locally on that one key. Storage: 100M URLs/month × ~500 bytes per record (long URL + short code + metadata) ≈ 50 GB/month, ≈ 3 TB over 5 years — trivially fits on modest hardware, so **storage is not the bottleneck here, read throughput and redirect latency are.** Code space: base62 (`[a-zA-Z0-9]`) with 7 characters gives 62^7 ≈ 3.5 trillion combinations — far more than needed for decades of growth at this write rate.

**API design**
- `POST /api/v1/shorten` — body `{long_url, custom_alias?, expires_at?}` → `{short_code, short_url}`
- `GET /{short_code}` → `302 Found` with `Location: <long_url>` (302, not 301, so that expired/deleted links can be changed and so analytics-on-click stays possible later — a permanent 301 gets cached by browsers and bypasses your server on repeat visits)
- `DELETE /api/v1/{short_code}` — owner-only deactivation

**High-level design**
Client → Load Balancer → stateless API servers. **Write path:** an API server requests a unique ID from an ID-generation service, base62-encodes it into a short code, and writes `{short_code → long_url}` into a key-value store. **Read path:** an API server checks a Redis cache for the short code first; on a hit, it issues the redirect immediately; on a miss, it falls back to the datastore, populates the cache, then redirects. Because reads dominate by ~100:1 and are simple point lookups, a **CDN or edge cache in front of the redirect endpoint itself** is worth considering for very hot links, since a 302 response can be cached briefly at the edge and never even reach origin.

**Deep dive**
Two real strategies for generating the short code, and this is usually the crux of the interview:
- **Base62-encode an auto-incrementing ID.** Guarantees uniqueness by construction (no collision handling needed) but a naive single auto-increment counter is a write bottleneck and a single point of failure. Fix: pre-allocate **ranges** of IDs to each API server (server A gets IDs 1–1,000,000, server B gets 1,000,001–2,000,000, etc.) coordinated through a lightweight service like ZooKeeper or a Redis `INCR`-based range allocator — this is the same range-partitioning idea behind Twitter's Snowflake ID scheme. The downside: sequential IDs are trivially guessable/enumerable (`abc123`, `abc124`, ...) — a real concern for a service whose links might point at unlisted content. Mitigation: don't base62-encode the raw counter directly; XOR or bit-shuffle it with a fixed key first (a reversible permutation), so codes look random to an outside observer but the server can still decode them back into the sequential ID for O(1) lookup if desired, or just use the ID purely as a primary key and store an independently-random short code as a second column.
- **Hash the long URL** (MD5/SHA-256), truncate to the first N base62 characters. Pro: no centralized ID generator needed, any server can compute a code independently. Con: **truncated hashes collide** — with enough URLs, two different long URLs can produce the same 7-character prefix — so this approach requires a collision check-and-retry loop (hash, check if code taken by a *different* URL, if so append a salt/increment and rehash) on write, which the counter-based approach avoids entirely. This is the direct tradeoff to name out loud: hash-based is simpler infrastructure but pushes collision-handling into every write; counter-based needs a coordinated ID generator but never collides.

**Bottlenecks & scaling**
The datastore is a simple key-value access pattern (`short_code → long_url`), which is exactly the shape Phase 3's **consistent hashing** partitioning (3.3, 5.11) is built for — shard by `hash(short_code)` across many nodes, and a technology like DynamoDB or Cassandra fits well since there's no need for joins or cross-row transactions. A viral link creates a hot key against one shard/cache node — the Redis layer absorbs the overwhelming majority of this since RAM access is ~1,000x faster than a disk-backed datastore read (Phase 1.5), and if one key gets extreme enough, replicate that specific hot key across multiple cache nodes rather than relying on a single node to absorb all its traffic.

---

## 7.2 Design a rate limiter (service-level)

**Requirements**
Functional: limit how many requests a given client (user ID, API key, or IP) can make in a time window, and reject excess requests with `429 Too Many Requests` (ideally with a `Retry-After` header). Different tiers get different limits (e.g., free tier 100 req/min, paid tier 10,000 req/min). Non-functional: the limiter must work correctly across a **fleet of many service instances** behind a load balancer, must add minimal latency to every request, and must have a defined behavior when the limiter's own infrastructure is unavailable.

**Estimation**
Assume an API gateway fronting 50,000 QPS total traffic. If every request requires one check against the rate-limit store, that store needs to sustain 50,000 ops/sec minimum — well within a single well-provisioned Redis node's typical ceiling (Redis routinely handles 100K–1M+ simple ops/sec), so raw throughput isn't the hard part; **availability and correctness under concurrent access from many app servers is.**

**API design**
Not typically a public-facing API — it's middleware invoked on every request. Internally: `check_and_increment(key, limit, window) -> {allowed: bool, remaining: int, reset_at: timestamp}`. An admin-facing config surface: `POST /admin/rate-limits {tenant_id, limit, window_seconds}`.

**High-level design**
Client → Load Balancer → **N stateless API server instances**, each of which calls a **centralized Redis (or Redis Cluster) rate-limit store** before doing any real work for the request. The counter is keyed by something like `ratelimit:{user_id}:{window_bucket}`.

**Deep dive**
The core design question is *which algorithm* (Phase 5.6 covers all four in depth — apply them here):
- **Token bucket** — a bucket refills at a fixed rate and each request consumes a token; naturally allows short bursts up to the bucket size while enforcing a long-run average rate. Good default for most APIs, since a user issuing 5 requests in the same second after being idle for a minute shouldn't be punished.
- **Sliding window log** — store a timestamp per request, count entries within the trailing window. Perfectly accurate, but memory cost scales with request volume per key, which is expensive at high QPS.
- **Sliding window counter** — an approximation blending the current and previous fixed window counts, weighted by how far into the current window we are. This is the practical sweet spot: nearly as accurate as the log approach, with the flat O(1) memory of a fixed counter — the standard recommendation here, implemented as a Redis `INCR` + `EXPIRE` pair wrapped in a Lua script for atomicity (avoiding a race between the increment and the expiry-check).

**Why not just count in-process on each server?** Because a naive per-instance in-memory counter only sees the slice of traffic that lands on *that* instance. If a user's requests get load-balanced round-robin across 10 servers, a "100 req/min" limit enforced independently per-server actually permits up to 1,000 req/min in aggregate — the limiter needs **shared state across instances**, which is exactly why it's centralized in Redis rather than kept local.

**Fail-open vs. fail-closed:** if the Redis-backed limiter itself becomes unreachable, what happens to every request that depends on it? **Fail-open** (let requests through when the limiter can't be reached) prioritizes availability — the API keeps serving traffic, at the cost of losing protection exactly when it might matter most (e.g., during an incident that's also generating a traffic spike). **Fail-closed** (reject everything when the limiter is down) protects backend services from being overwhelmed but turns a rate-limiter outage into a full API outage. Most consumer-facing APIs choose fail-open with a circuit breaker (Phase 6.2) around the Redis call plus a rough local in-memory fallback cap as a safety net; systems where every request is expensive or dangerous (e.g., a paid third-party API call downstream) more often choose fail-closed.

**Bottlenecks & scaling**
The Redis store is a new single point of failure introduced specifically to solve the shared-state problem — mitigate with a Redis Cluster (sharded + replicated) rather than a single node. For extremely hot keys, add a short-TTL local cache of very recent allow/deny decisions on each API server to cut down Redis round-trips without meaningfully weakening enforcement. Note that any distributed rate limiter has minor inaccuracy at window boundaries under true sliding-log semantics — the sliding-window-counter approximation is the accepted cost of staying fast and cheap.

---

## 7.3 Design a key-value store (mini Dynamo)

**Requirements**
Functional: `put(key, value)`, `get(key)` over arbitrary blobs, no cross-key transactions or complex queries needed. Non-functional: this is explicitly an **AP system** in CAP terms (Phase 3.9) — availability and partition tolerance matter more than strict consistency, tunable per-operation, modeled directly on Amazon's Dynamo paper.

**Estimation**
Assume 1 billion keys, average value size 1 KB → 1 TB of raw data; with 3x replication for durability/availability, ~3 TB total. Assume 100,000 reads/sec and 20,000 writes/sec at peak — a workload profile common for session stores, shopping carts, and similar always-available metadata stores.

**API design**
- `PUT /keys/{key}` — body `{value, context}` (context carries version/causality info back from a prior read, needed for conflict resolution)
- `GET /keys/{key}` → `{values: [...], context}` — note **plural** values: on a conflict, the store may return multiple concurrent versions for the client/application to resolve
- `DELETE /keys/{key}`

**High-level design**
Client → any node in the cluster can act as **coordinator** for a request (no special "primary" node). The coordinator uses **consistent hashing** (Phase 3.3 / 5.11, applied directly and exactly here — this problem is the canonical use case) to determine which N nodes on the ring are responsible for the key, forwards the write to those N replicas, and waits for **W** acknowledgments before returning success to the client; reads query **R** replicas and reconcile. Cluster membership and failure detection propagate via a **gossip protocol** between nodes rather than a central coordinator.

**Deep dive**
- **Partitioning via consistent hashing with virtual nodes** (5.11): each physical node owns many small virtual positions scattered around the hash ring rather than one big contiguous arc, so that when a node joins or leaves, load redistributes evenly across the *remaining* nodes instead of dumping all of it onto one physical neighbor.
- **N/W/R quorums:** N = replication factor (e.g., 3), W = writes required to acknowledge before success, R = reads required before returning a result. Setting **W + R > N** (e.g., W=2, R=2, N=3) guarantees any read overlaps with the most recent write on at least one replica — a tunable quorum, not a hardcoded consistency level. Lowering W increases write availability at the cost of a higher chance of reading stale data; this tunability is the whole point of a Dynamo-style store versus a single-leader database.
- **Conflict resolution — vector clocks vs. last-write-wins:** because multiple replicas can accept writes concurrently (especially during a network partition), two versions of the same key can genuinely conflict rather than one simply overwriting the other. A **vector clock** — a per-key map of `{node_id: counter}` — tracks causal history well enough to detect "these two versions are actually concurrent, not one derived from the other," at which point the store returns *both* versions to the client on the next read and lets the application merge them (e.g., union two shopping-cart versions). **Last-write-wins (LWW)**, using a synchronized timestamp, is simpler and adequate when losing a concurrent update occasionally is acceptable — a real tradeoff to name explicitly, not just "use vector clocks because Dynamo did" (Phase 3.12 covers this same tradeoff generically).
- **Hinted handoff:** if one of a key's N designated replica nodes is temporarily down when a write arrives, the coordinator writes to a different, healthy node instead, tagging the write with a "hint" saying which node it was really meant for. Once the original node recovers, the hinted data is forwarded to it and removed from the temporary holder. This lets the system keep meeting its W quorum during a transient failure without blocking writes — a direct availability-over-consistency tradeoff, cleaned up later by **anti-entropy** (background comparison of replicas via Merkle trees, which let two replicas efficiently find *which* keys differ without transferring the entire dataset to compare).

**Bottlenecks & scaling**
Hot partitions are addressed by virtual nodes redistributing load on membership changes. Node failure is absorbed by hinted handoff plus **sloppy quorums** (accepting writes from any N reachable nodes rather than strictly the "correct" N when some are down) — an explicit availability-over-consistency choice, which is the entire philosophical stance of a Dynamo-style system versus a strongly consistent one.

---

## 7.4 Design a distributed cache

**Requirements**
Functional: `get`, `set` (with TTL), `delete`, sitting as a look-aside cache in front of a slower system of record. Non-functional: sub-millisecond latency within a datacenter, must survive individual node failure without a catastrophic cache-hit-rate collapse, must scale horizontally as working-set size grows.

**Estimation**
Assume a 500 GB working set and cache nodes with 64 GB of usable RAM each → at least ~8 nodes just to hold the data, realistically 12–16 with headroom for growth and uneven key distribution. Assume 200,000 ops/sec total across the cluster.

**API design**
A thin client library, not a network-facing REST API: `get(key)`, `set(key, value, ttl)`, `del(key)` — the library itself decides which physical node to talk to (see below), then speaks the target node's native protocol (e.g., Redis's RESP protocol).

**High-level design**
Client (app server) → client-side hashing library resolves the owning cache node for a key via **consistent hashing** → talks **directly** to that node (no proxy hop in between, to keep latency minimal) → on a cache miss, the app server itself queries the system of record (the database) and writes the result back into the cache (classic **cache-aside**, Phase 5.2).

**Deep dive**
- **Shard assignment via consistent hashing** (5.11, again the direct application): adding or removing a cache node only remaps roughly `1/N` of keys, versus a naive `hash(key) % N` scheme which reshuffles almost every key on any topology change — the entire cache would effectively empty and refill, hammering the database with a stampede of misses.
- **Client discovery of node ownership:** clients need to agree on the current ring topology without a central bottleneck on every request. Practically, either (a) a lightweight shared **metadata/coordination service** (ZooKeeper, etcd, or even a slowly-polled config endpoint) broadcasts ring membership changes to all client libraries, which then compute hash assignment locally, or (b) a thin proxy layer (e.g., Twemproxy) sits in front and does the routing centrally, trading a small latency hop for simpler client logic.
- **Eviction policy:** LRU is the sound general-purpose default — cheap to approximate (Redis's `allkeys-lru` uses sampling rather than a perfectly exact LRU list, trading a little precision for speed) and works well when "recently used implies likely to be used again" holds, which is true for most access patterns. LFU (`allkeys-lfu`) is a better fit specifically when access frequency is more predictive than recency — e.g., a small set of reference/config data that's read constantly but might not have been touched in the last few minutes, which pure LRU would wrongly evict.
- **Invalidation strategy:** TTL-based expiry is simplest (bounded staleness window, no coordination needed) but tolerates some staleness by design. Write-through (the write path updates cache and DB together, synchronously) eliminates that window but adds latency to every write and couples the two systems' availability. Explicit invalidation (the writer publishes an invalidate-this-key event via pub/sub to all cache nodes/app instances on write) removes stale data immediately without paying write-through's synchronous cost, at the price of needing a reliable pub/sub delivery path.

**Bottlenecks & scaling**
A single celebrity/hot key can overwhelm the one shard that owns it even though the rest of the cluster is fine — mitigate with a small local (in-process, on the app server) L1 cache for the hottest handful of keys, or explicitly replicate a hot key across several shards. A **cache stampede** — a popular key expiring and hundreds of concurrent requests all missing simultaneously and hammering the database at once — is addressed with a mutex/single-flight pattern (only one request actually goes to the DB, others wait for its result) or probabilistic early expiration (refresh slightly before actual TTL, staggered, so misses don't all land in the same instant).

---

## 7.5 Design a notification system

**Requirements**
Functional: deliver a notification to a user via one or more channels (push, email, SMS) triggered by some backend event; respect user channel preferences/opt-outs; avoid sending the same logical notification twice. Non-functional: push should land within seconds, email/SMS within minutes is acceptable; the system must decouple *notification generation* (many unrelated backend services triggering notifications) from *delivery* (which has its own retry/failure characteristics per channel).

**Estimation**
Assume 50M notifications/day → ~580/sec average, with realistic peaks (a broadcast event) reaching several thousand/sec. If the average notification fans out across 1.5 channels (many users have push + email enabled), that's ~870 channel-sends/sec average.

**API design**
Producer-facing (internal): `POST /notifications {user_id | topic, template_id, data, channels: [...], dedupe_key}`. `dedupe_key` is the caller-supplied idempotency key described below.

**High-level design**
Many producer services → Notification API → writes an event onto a **queue/log** (Kafka, partitioned by `user_id` so one user's notifications stay ordered) — this is the decoupling point: producers only need to successfully enqueue, not wait on slow external providers. Per-channel **worker pools** (push workers, email workers, SMS workers) consume from the topic independently, each calling its channel's provider: **APNs/FCM** for push, **SES/SendGrid** for email, **Twilio** for SMS.

**Deep dive**
This is a direct application of the **fan-out pattern from Phase 5.10**: for a broadcast notification going to a large audience (e.g., "a show you follow just aired"), fan-out-on-write pushes a per-recipient job onto the queue at generation time. Because notification generation volume is much lower than, say, a social feed's read volume, fan-out-on-write is almost always the right call here (fan-out-on-read doesn't really apply — there's no "pull my notifications lazily" equivalent to a feed timeline; delivery needs to be pushed).

**Deduplication:** compute an idempotency key as `hash(user_id, event_type, event_id)` and check-and-set it in Redis with a TTL matching the "don't repeat this within X" window *before* enqueueing the delivery job — this is the same idempotency-key mechanic from **Phase 5.12**, applied to prevent a retried or duplicated upstream event from producing two pushes for the same logical notification.

**Channel-specific reliability differs meaningfully:** push is inherently best-effort (a device can be offline, a token can be stale) — the system maintains a device-token registry per user (updated on login/token refresh) and simply drops undeliverable pushes rather than retrying indefinitely. Email and SMS providers, in contrast, support explicit retry-with-backoff and delivery-status webhooks, so those workers track delivery state and retry transient failures. A **priority queue split** matters too — a transactional notification (OTP/2FA code) needs to bypass a backlog of marketing broadcast notifications, so they're routed to logically separate queues with different worker pool priorities rather than sharing one FIFO.

**Bottlenecks & scaling**
External provider rate limits (APNs and FCM throttle per-app send rates) are the recurring constraint — each provider gets its own rate-limited send queue (the exact pattern from 7.2, applied per-provider instead of per-user). A single huge fan-out event (a celebrity broadcast) is absorbed by Kafka's partitioning — spreading the resulting per-user jobs across many partitions keyed by `user_id`, consumed in parallel by a horizontally-scaled worker fleet, rather than any single component processing the whole burst serially.

---

## 7.6 Design a web crawler

**Requirements**
Functional: starting from seed URLs, fetch pages, extract links, discover and crawl new pages, store content for downstream indexing. Non-functional: **politeness** (never overload a single domain's server), respect `robots.txt`, avoid re-crawling unchanged pages excessively, and avoid re-processing URLs already seen, all at a scale of billions of pages.

**Estimation**
Target: crawl 1B pages/month → ~400 pages/sec average. At an average page size of ~500 KB (HTML + inline resources counted loosely), that's ~200 MB/sec of fetch bandwidth at steady state. Raw storage: 1B pages × 500 KB ≈ 500 TB/month before compression, ~100–150 TB/month after typical HTML compression ratios.

**API design**
Almost entirely internal — no public API. An admin/seed interface: `POST /crawl/seed {url, priority}`.

**High-level design**
Seed URLs → **URL Frontier** (a prioritized, distributed queue structure) → a fleet of **Fetcher workers** pull URLs from the frontier and download pages → a **Parser** extracts outbound links and content → a **Dedup filter** checks "have we seen this URL before" → genuinely new URLs get pushed back into the frontier; fetched content is written to **blob storage (5.8)** and handed off to a downstream indexing pipeline (feeding the **inverted index** described in 5.9).

**Deep dive**
The frontier isn't one flat queue — it's structured in two levels specifically to balance priority against politeness. A **front-queue layer** ranks URLs by priority (freshness need, estimated page importance/PageRank-style score), feeding into a **back-queue layer** with **one queue per host**, so that pulling work always respects a strict per-domain rate (e.g., no more than 1 request/sec to any single domain) — this is exactly the rate-limiting problem from **7.2**, applied per-domain instead of per-user. Hosts are mapped to their queue via consistent hashing, so the same worker(s) consistently own a given domain's crawl state.

**Dedup at billion-page scale** can't use a plain in-memory hash set — 1B URLs at roughly 100 bytes each for a real hash-set entry overhead would need well over 100 GB just for the set, before any actual crawling infrastructure. A **Bloom filter** solves this: a probabilistic set representation using roughly 1–2 bytes per entry instead of ~100, at the cost of a small, tunable false-positive rate (occasionally re-skipping a URL that was never actually crawled) — an acceptable tradeoff at this scale, since missing a rare page is far cheaper than the memory cost of perfect dedup. The Bloom filter itself is sharded across crawler nodes alongside the frontier.

**robots.txt** is fetched and cached (with its own TTL) per domain *before* any page on that domain is crawled, and both its `Disallow` rules and `Crawl-delay` directive are honored directly by the per-host queue's pacing.

**Bottlenecks & scaling**
Recrawl frequency is tuned per page using an estimate of how often that specific page's content actually changes historically (a news homepage needs re-crawling far more often than a static reference page) rather than a single global recrawl interval — this keeps fetch capacity spent where it matters. Malicious or accidental crawler traps (infinite link generation on one domain) are capped with a maximum crawl depth and a maximum pages-per-domain limit. The frontier itself, sharded by domain hash across many machines, is the component that needs to scale alongside fetcher count — a single-machine frontier would become the bottleneck long before fetch bandwidth does.

---

## 7.7 Design a multi-tenant SaaS backend

**Requirements**
Functional: the same application backend serves many independent customer organizations ("tenants"), each with isolated data and its own usage quota. Non-functional: no tenant's traffic or data should be visible to or degrade the experience of another tenant (the **noisy neighbor problem**), and the system must support per-tenant rate limits distinct from any global limit.

**Estimation**
Assume 10,000 tenants ranging from 10 to 100,000 end users each, ~5M total end users, average baseline load ~2 req/sec/tenant, but the top 1% (100 large tenants) running ~10x that. Total steady-state QPS ≈ 15,000, heavily skewed toward a small number of large tenants.

**API design**
Same REST surface for every tenant, with the tenant resolved per-request via a subdomain (`acme.saasapp.com`) or a claim embedded in the auth JWT — every downstream call is implicitly scoped: `GET /api/v1/orders` resolves to "orders for the tenant identified by this request's credentials," never taking a raw tenant ID from an untrusted client parameter.

**High-level design**
Client → Load Balancer → API gateway resolves tenant identity from the JWT/subdomain → **per-tenant rate limiter** (the same Redis-backed limiter from **7.2**, keyed by `tenant_id` instead of `user_id`) → stateless application servers → a data layer whose isolation strategy depends on tenant tier (below).

**Deep dive**
Three real data-isolation strategies, and the interview-worthy part is naming the tradeoff of each rather than picking one dogmatically:
- **Shared database, `tenant_id` column on every table.** Cheapest to run and simplest to operate — one schema, one migration path, one set of connection pools. Weakest isolation: a missing `WHERE tenant_id = ?` clause is a real, catastrophic cross-tenant data leak, and one large tenant's query load can degrade a shared table's performance for every other tenant on it. Best fit for the long tail of small, low-traffic tenants.
- **Schema-per-tenant** (same database instance, separate schema namespace per tenant). Meaningfully stronger logical isolation than a shared table, but tenants still share the same underlying compute/IO/connection-pool resources, so noisy-neighbor CPU/IO contention isn't actually solved — and migrations now have to run once per tenant schema, which gets operationally heavy in the thousands.
- **Database-per-tenant.** Strongest isolation and blast-radius containment (one tenant's runaway query or corrupted data can't touch another's DB at all), enables true per-tenant backup/restore and even per-tenant scaling. Operationally the most expensive: connection-pool exhaustion risk grows with tenant count, and migrations must be orchestrated across potentially thousands of independent databases. Reserved for enterprise tenants who are both large enough and paying enough to justify dedicated resources.

The realistic answer for a real product at this scale is a **hybrid, tiered by tenant size**: the long tail of small tenants lives in a shared, sharded pool (sharded by `hash(tenant_id)`, so 7.7 is really Phase 3.3's partitioning applied with `tenant_id` as the partition key), while large/enterprise tenants get dedicated databases. This is exactly why `tenant_id` needs to be the partition key from day one — migrating a tenant from shared to dedicated later is a live-data-migration problem, not a config change.

Noisy-neighbor protection isn't only a data-layer concern — compute needs the same tiering, e.g. separate worker/connection pools (or Kubernetes namespaces with CPU/memory quotas) for enterprise vs. free-tier traffic, so a free-tier tenant's burst can't starve paying tenants' request queue.

**Bottlenecks & scaling**
A shared shard that happens to host one unexpectedly large tenant needs a clear escape hatch — detect via per-tenant load metrics, then migrate that tenant to its own dedicated shard/DB. Quota and rate-limit enforcement must happen at the gateway, before expensive downstream work starts, not after a query has already consumed shared resources.

---

## 7.8 Design Twitter/X (news feed + fan-out)

**Requirements**
Functional: post a tweet, follow/unfollow users, view a home timeline aggregating tweets from followed accounts, like/retweet. Non-functional: reads vastly outnumber writes, follower counts are extremely skewed (median user has ~100 followers; some accounts have 100M+), and timeline reads need to feel instant.

**Estimation**
Assume 300M daily active users, ~20% of whom post, averaging 2 tweets/day when they do → ~120M tweets/day → ~1,400 writes/sec average (peaking several times higher around major events). Assume each user checks their timeline ~5x/day → 1.5B timeline reads/day → ~17,000 reads/sec average, with realistic peaks well above 50,000/sec. That read:write ratio (~12:1 on these numbers, likely higher in practice once you account for infrequent posters checking their feed repeatedly) is the number that justifies precomputing timelines rather than computing them on every read.

**API design**
- `POST /tweets {text, media?}` → `{tweet_id}`
- `GET /timeline/home?cursor=` → paginated list of tweets
- `POST /follow/{user_id}` / `DELETE /follow/{user_id}`
- `GET /users/{id}/tweets` — a user's own tweet history

**High-level design**
Client → Load Balancer → stateless API servers. **Write path:** a new tweet is written to a Tweet store (Cassandra, partitioned so that a user's own tweets are co-located for fast profile-page reads) and, asynchronously, a **fan-out service** pushes the tweet's ID onto each follower's precomputed timeline list (a capped Redis list, holding roughly the most recent ~800 tweet IDs per user). **Read path:** `GET /timeline/home` reads directly from the requesting user's precomputed Redis list (O(1)-ish) and hydrates the actual tweet content from the Tweet store/cache.

**Deep dive**
This is the textbook application of **fan-out-on-write vs. fan-out-on-read from Phase 5.10**. Fan-out-on-write (push) precomputes every follower's timeline at post time, which is exactly right given the read:write skew above — it converts an expensive "aggregate tweets from everyone I follow" read into a cheap "read my precomputed list" operation, paid for by amortizing the cost onto the (much rarer) write. It breaks down hard for **celebrity accounts**: a single tweet from an account with 100M followers would trigger 100M individual fan-out writes — the classic "celebrity problem." The standard fix is a **hybrid**: accounts above a follower-count threshold (say, 1M+) are excluded from the write-time fan-out entirely. At read time, a user's timeline is assembled by merging their normal precomputed list (from non-celebrity follows) with a small, live, on-demand pull of recent tweets from the handful of celebrities they follow (most users follow only a few such accounts, so this pull stays cheap). This keeps the common case — the 99%+ of users who aren't celebrities — at O(1) precomputed reads, while bounding celebrity fan-out cost to zero write amplification at the cost of a small extra read-time merge.

**Bottlenecks & scaling**
Fan-out queue depth can spike hard during a major real-time event (many people tweeting, many with large-but-not-celebrity followings) — the fan-out workers are horizontally scaled, consuming from a Kafka topic partitioned by follower shard so the burst spreads across many consumers. Precomputed timeline storage is capped at a few hundred recent tweet IDs per user (older history is reconstructed on demand from the Tweet store rather than kept indefinitely in the hot cache) to bound Redis memory footprint. The Tweet store itself is sharded using **consistent hashing (Phase 3.3)**, with tweet IDs generated Snowflake-style (embedding a timestamp) so that range queries by recency stay cheap even across shards.

---

## 7.9 Design WhatsApp/Messenger (real-time chat, delivery guarantees)

**Requirements**
Functional: 1:1 and group messaging, message delivery status (sent/delivered/read), messages queue for offline recipients and deliver on reconnect. Non-functional: near-real-time delivery when both parties are online, end-to-end encryption so the server never sees plaintext, must scale to hundreds of millions of concurrent persistent connections.

**Estimation**
At WhatsApp's real scale: ~2B users, ~100B messages/day → ~1.15M messages/sec average, peaking to several million/sec. At ~100 bytes per text message body, that's roughly 115 MB/sec of message-body bandwidth alone at average load, before headers/encryption overhead. Assume ~500M users concurrently connected at peak; if each connection-handling server can hold ~50,000 concurrent WebSocket connections (a realistic figure for a well-tuned event-loop server, Phase 1.6), that requires on the order of **10,000 connection gateway servers** just to hold sockets open.

**API design**
Primarily a persistent WebSocket protocol rather than request/response REST. Conceptually: `sendMessage(to, body, client_msg_id)`, and server→client pushes of `messageDelivered(message_id)` / `messageRead(message_id)` acks.

**High-level design**
Client opens a persistent WebSocket to a **Connection Gateway** server (an L4 load balancer just needs to pick *any* available gateway at connect time, since routing individual messages afterward is a separate concern from initial connection placement). On connect, the gateway registers `user_id → gateway_server_id` in a **Presence/Connection Registry** (Redis). When user A sends a message to user B: A's gateway looks up B's owning gateway in the registry, then routes the message over an internal channel (Redis pub/sub, or a dedicated internal message bus) to B's specific gateway, which pushes it down B's live socket. If B is offline (no registry entry), the message is instead persisted to a durable **offline queue** and delivered on B's next connect.

**Deep dive**
The connection registry is the load-bearing piece of this whole design: with thousands of gateway servers, **no single gateway inherently knows which other gateway is holding a given recipient's socket** — this lookup-then-route step is the core mechanic every real-time chat system at scale needs, and it's the same conceptual registry pattern useful elsewhere connection ownership matters (7.10's driver-push notifications reuse the same idea).

**Delivery guarantees, precisely:** "sent" fires once the server has durably accepted the message from the sender; "delivered" fires once the recipient's client has ack'd receipt over its live socket; "read" fires once the recipient's client acks after actually displaying it in the UI. Each of these is a separate ack message routed back through the same registry-lookup path toward the original sender.

**Offline queuing:** undelivered messages are held in a durable per-recipient store (e.g., Cassandra, partitioned by recipient) until acked, then removed — this must support multiple devices per user, since a message might need holding for one of a user's devices while another has already received it.

**End-to-end encryption**, at the conceptual level worth naming in an interview: the industry-standard approach is the **Signal Protocol** — each device holds an identity key plus a set of pre-generated one-time "prekeys," and the **Double Ratchet algorithm** derives a fresh encryption key for every message sent, giving **forward secrecy** (compromising one message's key doesn't expose any past or future message). The server only ever relays ciphertext; it never possesses the keys or the plaintext, which is also *why* server-side content search or automated moderation on message text is architecturally impossible in a true E2E design, not merely a policy choice.

**Bottlenecks & scaling**
A gateway server failing drops every connection it was holding — clients detect this via a heartbeat/ping-pong and reconnect to a different gateway, which re-registers them in the presence registry. Group messages fan out to every member's respective gateway — the same fan-out-on-write concern from **Phase 5.10**, applied to real-time push instead of a feed — and for very large groups this becomes a genuine broadcast problem similar in shape to 7.5's notification fan-out.

---

## 7.10 Design Uber/ride-sharing (geo-indexing, matching)

**Requirements**
Functional: a rider requests a ride, gets matched to a nearby available driver; drivers continuously stream their location; both parties see live location during the trip. Non-functional: matching latency should be a few seconds at most, the system must sustain an extremely high volume of small, constant location-update writes, and it must handle demand spikes (rush hour, bad weather) gracefully.

**Estimation**
Assume 5M active drivers each pinging location every 4 seconds → ~1.25M location writes/sec system-wide — this single number dominates the entire design; everything else is secondary by comparison. Ride requests: assume 20M rides/day → ~230/sec average, with rush-hour peaks realistically 5x that, ~1,150/sec. Live driver location state is small and ephemeral (5M drivers × ~100 bytes ≈ 500 MB) and fits comfortably in memory — it's the *write rate*, not the data volume, that's the hard constraint.

**API design**
- `POST /drivers/{id}/location {lat, lng, timestamp}` — high-frequency, from the driver app
- `POST /rides/request {pickup, dropoff}` → match status, driver info once matched
- A WebSocket/push channel delivering live driver-location updates to the rider during matching and the trip itself

**High-level design**
Driver apps → a lightweight, very high-throughput **Location Ingestion service** → writes into an in-memory **Geo-Index** (geohash- or quadtree-based). When a rider requests a ride, the **Matching service** queries the geo-index for nearby available drivers within a radius, ranks candidates (distance, ETA, acceptance rate/rating), and sends a match offer to the top candidate via the same connection-registry-and-push mechanism described in **7.9**. On acceptance, a ride record is created in a relational **Ride DB** (the actual source of truth for trip and payment data, where strong consistency matters).

**Deep dive**
**Geospatial indexing — two real options:**
- **Geohashing** encodes latitude/longitude into a base32 string where a shared prefix implies physical proximity, which makes "find nearby drivers" a simple prefix-range query and makes sharding the index by geohash prefix natural (a direct cousin of consistent-hashing-style partitioning by key). The catch: two points can be geographically adjacent yet fall into different geohash cells if they straddle a grid boundary, so a real implementation must also check neighboring cells, not just the exact prefix match.
- **A quadtree** recursively subdivides space into four quadrants, but only where driver density actually requires finer resolution — dense in cities, coarse in rural areas. This adapts naturally to ride-sharing's real distribution (drivers cluster heavily in urban cores), at the cost of more complex tree-rebalancing logic compared to geohash's flat string comparisons.

In practice, an in-memory store like **Redis with `GEOADD`/`GEORADIUS`** (which is itself geohash-based under the hood, encoded into sorted sets) is the standard pragmatic choice: the live driver-location dataset is small enough to fit in RAM, and Redis comfortably sustains the write rate this problem demands with low-latency radius queries.

**Matching mechanics:** a candidate driver is offered a ride and given a short acceptance window (e.g., 10–15 seconds); on timeout or decline, the offer cascades to the next-ranked candidate. This offer/timeout/reassign cycle needs a claim-and-lease mechanism structurally identical to the job-claiming pattern in **7.16** — the offer is effectively a lease that expires if unclaimed.

**Bottlenecks & scaling**
The 1.25M writes/sec of location pings is the dominant load, and it doesn't need to be applied to the shared geo-index at full fidelity in real time — updating the index every 2–4 seconds per driver (rather than on every single raw ping) is sufficient for matching purposes, while the full raw ping stream is still captured via Kafka for analytics/ETA-model training. The geo-index itself is sharded by region/geohash prefix across many nodes, since no single Redis instance can absorb global location-update traffic at this rate. The Matching service is stateless and scales horizontally; the shared geo-index state is what must scale in lockstep with it.

---

## 7.11 Design YouTube/Netflix (video storage + streaming + CDN)

**Requirements**
Functional: upload a video, transcode it into multiple resolutions/bitrates, stream it back with adaptive quality based on the viewer's network, browse/search a catalog. Non-functional: uploads and transcoding can tolerate minutes of latency; playback start and quality switches must feel instant; the system serves an enormous, long-tail catalog where a small fraction of content gets nearly all the views.

**Estimation**
Using YouTube-scale ballpark numbers: ~500 hours of video uploaded per minute, at roughly 1.5 GB/hour for a reasonably compressed source upload, is ≈ 750 GB/minute of raw ingest ≈ ~1 PB/day. After transcoding into ~5 renditions spanning 240p to 4K, total stored output multiplies to roughly 3–4 PB/day across all renditions combined. Playback volume (billions of views/day) is overwhelmingly absorbed by CDN edge caching rather than hitting origin storage directly.

**API design**
- `POST /videos/upload` → returns a pre-signed, direct-to-blob-storage multipart upload URL (Phase 5.8) so raw video bytes never transit the application servers themselves
- `GET /videos/{id}/manifest` → an adaptive-streaming manifest (HLS `.m3u8` or DASH `.mpd`)
- `GET /videos/{id}/metadata` → title, description, available renditions

**High-level design**
Client uploads directly to **Blob Storage (5.8)** via a pre-signed multipart URL, bypassing app servers for the heavy bytes. Upload completion fires an event that enqueues a **transcoding job**; a distributed worker fleet pulls jobs, produces multiple resolution/bitrate renditions, chunks each into small (2–10 second) segments, and writes them back to blob storage. A **CDN (Phase 5.3)** pulls and caches segments at edge locations on first request — deliberately **pull-based** rather than push-to-every-edge, because the catalog is enormous and most videos are rarely watched (a long tail), so proactively pushing everything everywhere would waste enormous edge capacity on content nobody's requesting there. A separate **Metadata DB** tracks `video_id → title/description/rendition list/manifest location`, entirely decoupled from where the actual bytes live.

**Deep dive**
The **transcoding pipeline** is the expensive, slow stage — a distributed worker fleet (often GPU-accelerated or using dedicated media-encoding hardware) pulls jobs from a queue, with each job scoped to a single `(video, target rendition)` pair so all renditions of one video transcode in parallel rather than serially. **Adaptive bitrate streaming (ABR)** works entirely client-side: the player continuously monitors its own measured network throughput and switches between pre-encoded bitrate renditions **segment by segment** — there's no server-side re-encoding happening live. The manifest file (HLS or DASH) simply lists every available rendition and its segment URLs, and the player's own logic decides which rendition to request next. Chunking into small segments matters for two independent reasons: it's the granularity at which ABR switching happens, and small immutable chunks are exactly what a CDN caches most efficiently, versus trying to cache (and partially serve, via range requests) one enormous file.

**Bottlenecks & scaling**
Upload spikes create transcoding backlog — absorbed by a horizontally-scaled worker pool pulling from a queue rather than transcoding synchronously on upload. The long tail of rarely-watched content isn't worth pre-transcoding into every rendition eagerly at upload time — a common optimization is to eagerly transcode only the most commonly requested renditions (e.g., 480p/720p) and lazily transcode higher/lower renditions on first actual request, or move cold content to cheaper storage tiers. A CDN cache miss on unpopular content falls through to origin storage with higher latency, which is an acceptable tradeoff since it's rare by construction.

---

## 7.12 Design Instagram (media storage, feed ranking)

**Requirements**
Functional: post a photo/video, follow other users, view a home feed, like/comment. Non-functional: similar read/write skew to Twitter, but with heavier media payloads driving storage/CDN concerns, and — the key difference from 7.8 — the home feed is **ranked**, not strictly chronological.

**Estimation**
Assume 500M DAU, 100M posts/day → ~1,150 writes/sec average for new posts. Feed reads dominate: assume 500M users refreshing their feed ~10x/day → 5B reads/day → ~58,000 reads/sec average, peaking to ~170,000/sec. Media averages roughly 200 KB for a photo, more for video, blending to ~2 MB/post → 100M × 2 MB = 200 TB/day of raw media ingest before any downstream compression/resizing.

**API design**
- `POST /media/upload` → pre-signed blob upload URL (same pattern as 7.11)
- `POST /posts {media_id, caption}`
- `GET /feed?cursor=`

**High-level design**
The delivery backbone mirrors **Twitter's (7.8)**: post metadata plus a fan-out of `post_id` into each follower's precomputed feed cache (Redis, capped list), with the same celebrity-threshold hybrid fallback. The meaningful difference is that **media bytes never sit inline** — they go to Blob Storage + CDN, exactly the pattern from **7.11/5.8/5.3**, and the feed cache only ever stores lightweight references (post IDs), never media payloads.

**Deep dive**
Delivery mechanics (fan-out-on-write with the celebrity hybrid, per **5.10**) answer "which posts are even *candidates* to show this user" — but that's a distinctly separate concern from **what order they should appear in**. A dedicated **ranking/scoring service** takes the candidate set (typically the last several hundred chronological `post_id`s pulled from the fan-out cache) and scores each one using a machine-learning model — features like recency, the viewer's past engagement with this specific poster, content type, and predicted like/comment probability — before returning the final ordered feed. This is deliberately kept out of deep-dive detail here: the interview-relevant point is knowing to name ranking as its **own pipeline stage with its own infrastructure** (a feature store, a model-serving layer) sitting *after* candidate generation and *before* the response is returned, rather than conflating "how posts reach a user's candidate pool" (a distributed-systems fan-out problem) with "what order they display in" (an ML-serving problem entirely orthogonal to it).

**Bottlenecks & scaling**
Media storage and bandwidth dominate cost and scale far more than for a text-first product like Twitter — mitigated by generating multiple resolution variants at upload time (thumbnail, feed-resolution, full-resolution) so a feed view never has to pull a full-resolution image just to render a small preview, plus aggressive CDN caching (5.3). The ranking service adds real read-path latency on top of a pure delivery system — mitigated by re-ranking only a bounded candidate window (a few hundred posts, not a user's entire history) and briefly caching a freshly-ranked result rather than recomputing it on every single pull-to-refresh.

---

## 7.13 Design Google Drive/Dropbox (file sync, conflict resolution)

**Requirements**
Functional: keep files in sync across a user's devices, handle edits made offline on multiple devices, retain version history, support sharing. Non-functional: syncing a small edit to a huge file shouldn't require re-uploading the whole file; conflicting concurrent edits must be handled predictably, not silently dropped.

**Estimation**
Assume 50M users averaging 5 GB stored each → ~250 PB total stored. Assume an average user makes 20 file-change events/day → 1B change events/day → ~11,500/sec average. Using Dropbox's real chunking approach as a reference point, a typical chunk size is ~4 MB, so a 1 GB file decomposes into ~250 chunks.

**API design**
- `POST /files/{path}/chunks {chunk_hashes: [...]}` → returns which of those hashes the server doesn't already have (the dedup check)
- `PUT /chunks/{hash}` — upload only the chunks the server reported missing
- `GET /files/{id}/manifest` → ordered list of chunk hashes composing the current version
- `GET /changes?since=cursor` — delta sync: what changed since the client's last known state

**High-level design**
A client watches the local filesystem for changes. On a change, it splits the affected file into chunks, hashes each (SHA-256), and asks the **Metadata Service** which of those hashes are already known (deduplication — including cross-user dedup, since two users storing an identical file only need the underlying chunks stored once). Only new/changed chunks get uploaded to **Blob Storage (5.8)**. The Metadata Service — a system entirely separate from where the bytes physically live — records the file's new **manifest** (its ordered chunk-hash list) as a new version, then notifies the user's other devices (via a push/registry mechanism structurally similar to 7.9's) that a change is available, so they can pull just the new chunks and reconstruct the file locally.

**Deep dive**
**Chunking for efficient sync:** splitting a file into ~4 MB blocks means editing a small part of a large document only requires re-uploading the handful of chunks that actually changed, not the entire file. **Content-defined chunking** (using a rolling hash, e.g. a Rabin fingerprint, to decide chunk boundaries based on content rather than fixed byte offsets) is more robust than naive fixed-offset chunking: inserting even a single byte near the start of a file would shift every subsequent fixed-offset boundary, making every later chunk falsely appear to have changed even though its actual content is identical — content-defined boundaries "re-sync" after an insertion and correctly identify that most chunks are unchanged.

**Conflict resolution when the same file is edited offline on two devices:** for opaque files (a `.zip`, a photo, most binary formats), the pragmatic and widely-used approach — Dropbox, and classic Drive for non-native files — is **last-write-wins on the canonical version, with the losing version preserved as a separate "conflicted copy" file** rather than silently discarded. This deliberately pushes resolution to the human, because arbitrary binary content has no general-purpose way to auto-merge two divergent versions. For **structured, real-time collaborative documents** (Google Docs specifically), the answer is fundamentally different: **Operational Transformation (OT)** or **CRDTs (Conflict-free Replicated Data Types)** let concurrent edits merge automatically at the level of individual operations (insert-at-position, delete-range), because the data model understands edit semantics rather than treating the document as an opaque blob. Worth naming explicitly as the correct answer for that narrower sub-case, but a materially different problem from whole-file sync — don't conflate the two if an interviewer asks about "Google Docs" specifically versus "Drive" generally.

**Bottlenecks & scaling**
The Metadata Service is the hot path for every single sync operation — checking and updating manifests — even though it moves far less raw data volume than blob storage; it needs to scale independently, typically as a sharded relational or document store keyed by file/user ID. Cross-user chunk-level dedup introduces a reference-counting requirement: a chunk can't be deleted from blob storage while *any* file manifest anywhere still references it, which requires a periodic garbage-collection sweep over reference counts rather than immediate deletion on a single file's delete.

---

## 7.14 Design a payment system (idempotency, exactly-once semantics)

**Requirements**
Functional: charge a customer, record the transaction in an accurate ledger, integrate with an external payment processor, notify the user of success/failure. Non-functional: **never double-charge on a retried request**, the ledger balance must be strongly consistent and accurate at all times, the rest of the pipeline (processor call, notification) can be eventually consistent.

**Estimation**
Assume 1M transactions/day → ~12/sec average, with a realistic peak event (e.g. a high-traffic sale) at ~20x, ~230/sec. Each transaction involves an external processor call at 200–500ms latency, a ledger write that needs to stay fast (<50ms, since it's on a strongly-consistent hot path), and an async notification that can tolerate seconds of delay.

**API design**
- `POST /payments {idempotency_key, amount, currency, source}` → `{payment_id, status}`
- `GET /payments/{id}` → current status

**High-level design**
Client → API server → **first** checks the `idempotency_key` against a dedup store before doing anything else. If new, the server writes an initial "pending" row to the **Ledger DB** *and* records the idempotency key **in the same local transaction**, then publishes an event via a **transactional outbox**. An async worker consumes the outbox, calls the external **Payment Processor**, and on response updates the ledger row to "completed"/"failed," then publishes a completion event that a separate consumer turns into a user notification (reusing **7.5's** notification system).

**Deep dive**
**Idempotency keys (Phase 5.12, applied directly):** the client generates one key per *logical* payment attempt — not per HTTP call — and resends the same key on every retry of that attempt. The server checks the key before making any processor call; if it's already been processed, it returns the previously recorded result instead of charging again. This is the direct, load-bearing defense against the canonical failure mode: a client's request times out waiting for a response, retries, but the *first* attempt actually succeeded server-side — without an idempotency key, the retry would trigger a second real charge.

**Why not a distributed transaction (2PC) across processor + ledger + notification (Phase 3.11):** the payment processor is a third-party system outside your control and simply cannot participate in a two-phase-commit protocol at all. Even setting that aside, 2PC's blocking nature — a coordinator crash mid-protocol leaves participants holding locks indefinitely — is unacceptable for a system that needs to keep accepting new payments continuously. The realistic pattern instead is the **transactional outbox + saga**: the initial DB write (ledger "pending" row + outbox event) commits atomically as one local transaction; a separate process drives every downstream step, and each step is individually **compensable** rather than part of one atomic cross-system commit. If the processor call fails, the compensating action is simply marking the ledger row "failed" — there's no charge to "undo," since it never externally succeeded.

**Reconciliation is not optional here, it's the actual safety net that makes eventual consistency acceptable for money.** A recurring batch job (nightly, or more frequent for high-volume systems) pulls the processor's own transaction records and diffs them against the ledger's "completed" rows, catching the genuinely possible case where a processor charge succeeded but the callback confirming it was lost before the ledger got updated — an async external call can fail silently in exactly this way, and reconciliation is what catches it after the fact rather than preventing it upfront.

**Strong consistency specifically for the ledger/balance:** the ledger is exactly the kind of data that cannot tolerate the AP-style eventual consistency used elsewhere in this curriculum's KV-store examples (7.3) — it needs a database with real ACID transactions (relational, single-leader for a given account's shard) so that "check balance, deduct, record" happens atomically and never double-applies against a concurrent deduction on the same account.

**Bottlenecks & scaling**
The processor call is the slow, externally-latency-bound step — it's handled fully asynchronously via the outbox/worker pattern specifically so it never blocks the API's response to the client. The ledger DB is sharded by `account_id` (not by transaction ID), so all operations touching one account's balance stay on a single shard/leader — preserving strong consistency exactly where it's needed, without forcing every unrelated transaction system-wide onto one global lock.

---

## 7.15 Design a search autocomplete/typeahead system

**Requirements**
Functional: given a prefix typed so far, return the top-K matching suggestions ranked by popularity, updating as the user types. Non-functional: must feel instantaneous per keystroke — this single requirement drives nearly every design decision here more than any other.

**Estimation**
Latency budget: to feel instant while typing, the round trip needs to land well within ~50–100ms. Assume 500M full searches/day feed the popularity signal, but suggestion *requests* fire far more often than searches since every keystroke can trigger one (even with debouncing, easily 5–10x the search volume) → roughly 30,000–60,000 suggestion requests/sec at peak.

**API design**
`GET /autocomplete?q={prefix}&limit=10` → `[{suggestion, score}, ...]`, called from the client on a debounced timer (~100–150ms after the last keystroke) rather than on every single keystroke, to avoid firing a request per character typed in rapid succession.

**High-level design**
Client fires a debounced request per pause in typing → Load Balancer → stateless API servers, each backed by (or querying) an **in-memory Trie** built from historical query frequency → a trie lookup walks to the prefix's node and returns its pre-sorted top-K children → results for extremely common short prefixes (`"a"`, `"how"`, `"wh"`) are additionally cached at an edge/CDN layer or a local cache, since those see disproportionate request volume relative to the number of distinct results they can return.

**Deep dive**
**Trie mechanics:** each node represents one character; critically, nodes **precompute and store their own top-K most frequent complete queries within their subtree** at index-build time, rather than computing that ranking live at query time. This precomputation is what keeps p99 latency in single-digit milliseconds — a query for a prefix is a single tree-walk plus an O(K) return, not a live scan or sort over however many completions exist under that prefix.

**Keeping the trie current without a full rebuild:** query popularity is a slowly-shifting signal in aggregate, but a breaking-news-style trending spike needs to surface within minutes, not hours — waiting for the next scheduled full rebuild would be far too slow. The practical answer is a split resembling **Phase 4.4's lambda-architecture pattern**: a **base trie** is rebuilt periodically (every few hours) from a batch job aggregating historical query logs, while a separate, much smaller **fast layer** — an in-memory counter/trending structure updated continuously by a stream processor consuming live search-query events — tracks very recent spikes and gets merged into or overlaid on the base trie's results at query time, boosting anything trending hard in just the last few minutes. Rebuilding the base trie wholesale, rather than mutating it in place, is a deliberate choice: in-place updates to a structure that's both heavily read under strict latency requirements and concurrently written would be a locking/correctness headache disproportionate to the benefit — instead, the new trie is built off to the side and swapped in atomically (a blue/green pointer swap) once ready.

**Bottlenecks & scaling**
A trie covering a truly global query corpus can exceed a single server's memory — partitioned by prefix range (e.g., `"aa"`–`"am"` on shard 1, and so on) across multiple trie-serving shards behind a thin routing layer. Hot short prefixes get their small top-K result set cached directly at the CDN/edge, since that result changes on the order of hours, not seconds — this alone can divert the large majority of single-character-prefix traffic away from the origin trie servers entirely.

---

## 7.16 Design a distributed job scheduler

**Requirements**
Functional: schedule one-off jobs (run at a specific time) and recurring jobs (cron-like expressions), execute each due job exactly once despite worker failures, scale to many concurrent workers and many scheduled job definitions. Non-functional: a worker crashing mid-job must not silently lose the job, and it must not cause the job to run twice unrecoverably either — practically, this problem accepts **at-least-once** execution with idempotent handlers, not true exactly-once.

**Estimation**
Assume 1M scheduled job definitions producing roughly 50,000 executions/hour in aggregate → ~14/sec average — but cron-style schedules cluster heavily at minute/hour boundaries, so momentary peaks (tens of thousands of jobs all becoming due in the same second at `:00`) can be 50–100x the average rate.

**API design**
- `POST /jobs {schedule: cron_expr | run_at, payload, retry_policy}` → `job_id`
- `DELETE /jobs/{id}`, `GET /jobs/{id}/status`
- Worker-facing: `POST /jobs/claim` (long-poll/lease a batch of due jobs), `POST /jobs/{id}/heartbeat`, `POST /jobs/{id}/complete`

**High-level design**
Job definitions live in a **Jobs DB** (relational, with an indexed `next_run_at` column). A **Dispatcher** process periodically scans for jobs due to run and pushes them onto a partitioned **work queue**. A fleet of **Workers** claims jobs from the queue; on claim, a job enters an "in-progress" state under a **lease/visibility timeout**; the worker heartbeats periodically to extend the lease while it works, then marks the job complete. For recurring jobs, the Dispatcher computes and writes the job's next `next_run_at` back to the Jobs DB after a successful run.

**Deep dive**
The **claim/lease mechanism is directly analogous to SQS's visibility timeout** (foreshadowed again in 7.17): claiming a job makes it invisible to other workers for a lease duration (e.g. 60 seconds); the worker must complete the job or heartbeat to extend the lease before it expires. If a worker crashes mid-job and stops heartbeating, the lease simply expires on its own and the job becomes claimable again — this is what gives worker-failure recovery a clean path *without* needing to explicitly detect that a worker died: the system only needs the lease timer, not a failure detector. The tradeoff this creates is **at-least-once, not exactly-once, execution** — it's possible for the original worker to be alive-but-slow and complete the job right after a second worker has already re-claimed and started it. Job handlers therefore need to be idempotent (Phase 5.12, again), or the claim needs a **fencing token** — a monotonically increasing lease ID that the job's side effects check before committing, so a stale worker's late completion can be detected and rejected rather than silently applied twice.

That lease-expiry mechanism handles one worker crashing mid-job. A **separate** problem is preventing *the Dispatcher itself*, if run as multiple replicas for availability, from double-enqueuing the same scheduled firing. This is solved either via **leader election** (only one active Dispatcher replica scans and enqueues at a time, coordinated through a consensus-backed lock such as ZooKeeper or etcd — Phase 3.10's consensus algorithms are exactly the mechanism underneath this) or via an atomic **compare-and-set update at the DB level** (`next_run_at` is only advanced, claiming the firing, if it hasn't already been claimed by a concurrent scan) — which lets multiple Dispatcher replicas scan concurrently without any coordination service at all, at the cost of relying on the DB's own atomic-update guarantee instead.

**Bottlenecks & scaling**
The "thundering herd at minute boundaries" problem — large numbers of cron jobs scheduled for `:00` all becoming due at once — is mitigated by jittering the actual dispatch of non-time-sensitive jobs by a few seconds, and by partitioning the queue and worker pool broadly enough that the burst spreads across many consumers rather than landing on one. The Jobs DB's "what's due right now" scan needs its `next_run_at` index and, once job-definition count is large enough, sharding of the Jobs DB itself (by `job_id` hash) so a single Dispatcher scanning one table doesn't become the bottleneck on its own.

---

## 7.17 Design Azure-flavored variants: a Blob Storage clone and a queue service

This pairing is worth practicing specifically for Microsoft-style interviews (per **Phase 0.5**) — even without needing Azure-specific product knowledge, "design a blob store" and "design a queue service" map directly onto **Azure Blob Storage** and **Azure Service Bus/Queue Storage**, and interviewers there often expect comfort with the underlying mechanics regardless of which vendor's name gets used.

### Part A — "Design S3" (a blob storage clone)

**Requirements**
Functional: store and retrieve arbitrary-size objects by `bucket + key`, support multipart upload for large objects, generate time-limited signed URLs for temporary access without exposing permanent credentials (Phase 5.8, directly). Non-functional: very high durability — S3's own stated target of 99.999999999% ("11 nines") is the reference number worth citing — and availability across node/rack/datacenter failures.

**Estimation**
Example scale: 10 PB stored, ~100,000 requests/sec blended across GET/PUT.

**API design**
- `PUT /{bucket}/{key}` (small objects) or a pre-signed direct-to-storage URL
- `GET /{bucket}/{key}`
- `POST /{bucket}/{key}?uploads` — initiate multipart upload
- `PUT /{bucket}/{key}?partNumber=N&uploadId=X` — upload one part
- `POST /{bucket}/{key}?uploadId=X&complete` — finalize, assembling all parts into one logical object

**High-level design**
For small objects, the API layer streams bytes directly to a **Storage Node** cluster. For large objects, a client uses multipart upload, splitting the object into (e.g.) 100 MB parts uploaded in parallel directly to storage nodes. A **Metadata Service** — deliberately separate from the storage nodes holding actual bytes — maps `bucket/key → physical location(s)`, and (during multipart upload) tracks which parts have arrived until the final "complete" call assembles them into one manifest. On `GET`, the API layer resolves location via the metadata service, then streams bytes from the correct storage node(s), optionally routed through a **CDN (5.3)** for frequently-accessed or public objects.

**Deep dive**
Durability comes from one of two mechanisms. **Replication** — store 3 full copies across separate failure domains (racks/availability zones) — is simple to reason about but costs 3x storage. **Erasure coding** splits an object into *k* data fragments plus *m* parity fragments spread across *k+m* nodes, such that any *k* of the *k+m* fragments are sufficient to reconstruct the original — e.g., 10 data + 4 parity fragments survives any 4 simultaneous node failures at only ~1.4x storage overhead instead of 3x. Real object stores (including S3's infrequent-access tiers) favor erasure coding at large scale specifically because the storage-cost savings compound enormously at petabyte scale, accepting somewhat higher CPU cost and reconstruction latency on failure as the tradeoff. **Multipart upload** exists because a single giant PUT over one TCP connection is fragile (any mid-transfer failure means restarting a multi-gigabyte upload from scratch) and slow (no parallelism); splitting into independently-uploadable, independently-retryable parts fixes both, with the final "complete" call telling the metadata service to assemble the manifest referencing every part as one logical object.

### Part B — "Design SQS/Service Bus" (a queue service)

**Requirements**
Functional: `SendMessage`, `ReceiveMessage`, `DeleteMessage`; optional FIFO ordering; automatic dead-letter handling for messages that repeatedly fail processing. Non-functional: **at-least-once** delivery (not exactly-once), configurable message retention (e.g., up to 14 days).

**Estimation**
Assume 500,000 messages/sec across the platform, ~10 KB average message size.

**API design**
`SendMessage(queue, body)` → `message_id`; `ReceiveMessage(queue, max_messages, visibility_timeout)` → messages with receipt handles; `DeleteMessage(queue, receipt_handle)`; a redrive policy (`maxReceiveCount`) configures automatic dead-letter routing.

**High-level design**
Producer → `SendMessage` → durably appended into the queue's backing store (a partitioned, replicated log or queue store). Consumer calls `ReceiveMessage`, which atomically marks a batch of messages invisible for the visibility-timeout window and returns them with receipt handles. The consumer processes the message and calls `DeleteMessage` before the timeout expires; if it doesn't, the message becomes visible again for redelivery — the exact same lease mechanic as **7.16's** job-claiming system.

**Deep dive**
At-least-once (never exactly-once) is the deliberate guarantee, not an accidental limitation: a message is only removed after an explicit delete, so any failure between receive and delete — a consumer crash, a lost network call on the delete itself — results in redelivery. This is precisely why consumers must be built idempotent from the start, the same recurring theme as **5.12**, **7.14**, and **7.16**. The **dead-letter queue** mechanism tracks a receive count per message; once it exceeds `maxReceiveCount` without a successful delete, the message is automatically moved to a configured DLQ instead of being redelivered indefinitely — this isolates "poison messages" (ones that crash every consumer that touches them) so they stop blocking the healthy flow of the main queue, while still preserving them for an operator to inspect and manually replay later.

**Bottlenecks & scaling**
Queue storage partitions across many nodes by queue name (or further by message hash for a single very-high-throughput queue) to scale write throughput horizontally. Strict FIFO ordering guarantees, when required, fundamentally cap parallelism within one ordering group — the same tension as Kafka's per-partition ordering guarantee from **Phase 4.2** — so FIFO should only be turned on for the specific queues that actually need strict order, since it measurably reduces achievable throughput versus a standard best-effort-ordered queue.

**Why this pairing specifically:** Blob Storage and Service Bus/Queue Storage are two of the most conceptually distinct primitives in Azure's platform (durable object storage vs. ephemeral message delivery), and being able to reason about the actual mechanics underneath both — partitioning, durability tradeoffs, delivery semantics — rather than just being able to name the products, is exactly the kind of "comfortable with the concepts behind the Azure services" comfort that **Phase 0.5** flags as the realistic bar for a Microsoft system design loop.
