# Phase 1 — Foundations

Everything in system design sits on top of these mechanics. This file goes deep on purpose — surface-level "X is used for Y" answers fall apart the moment an interviewer asks "why" or "how does that actually work."

---

## 1.1 How computers talk: client-server, IP, TCP vs UDP, sockets

**Client-server model:** a client initiates a request, a server responds. Almost every system design problem is really "how do we scale this basic pattern to millions of clients." The layered way to think about it is the **OSI-style mental model** (simplified to what actually matters day to day):

- **Application layer** — HTTP, DNS, SMTP, etc. — what your code actually speaks.
- **Transport layer** — TCP or UDP — how data gets from process to process reliably (or not).
- **Network layer** — IP — how data gets from machine to machine across networks.
- **Link/physical layer** — Ethernet, WiFi — how bits actually move over a wire or radio.

**IP (Internet Protocol):** every machine has an address (IPv4 like `192.168.1.1` — 32-bit, ~4.3 billion addresses, effectively exhausted; IPv6 — 128-bit, astronomically larger, adopted specifically to solve address exhaustion). Data travels as **packets**, routed hop-by-hop across routers, each of which only knows "send this toward the next router closer to the destination" — no single router knows the whole path. IP itself makes **no promise** a packet arrives, or arrives in order, or arrives only once. That's the whole reason TCP exists on top of it.

**TCP (Transmission Control Protocol) — how it gets reliability out of an unreliable network:**
- **Three-way handshake to open a connection:** client sends `SYN` (synchronize, "I want to talk, here's my starting sequence number"), server responds `SYN-ACK` (acknowledges + sends its own sequence number), client responds `ACK`. Only after this does either side send actual data. This is also why TCP has connection setup latency that UDP doesn't.
- **Sequence numbers + acknowledgments:** every byte sent is numbered. The receiver acks which bytes it has received; unacked bytes get retransmitted after a timeout. This is how out-of-order or lost packets get detected and fixed transparently to the application.
- **Four-way close:** `FIN` → `ACK` → `FIN` → `ACK` (each side closes its own direction independently, which is why you'll sometimes see a `TIME_WAIT` state — the closing side waits to make sure the last ACK actually arrived).
- **Flow control & congestion control:** TCP dynamically adjusts how much data it sends before waiting for an ack, both to avoid overwhelming a slow receiver (flow control) and to avoid overwhelming the network itself (congestion control — this is why TCP throughput ramps up gradually at the start of a connection instead of blasting at full speed immediately).

**UDP (User Datagram Protocol) — deliberately none of that:**
- No handshake, no sequence numbers, no retransmission, no ordering guarantee. You send a packet ("datagram") and it either arrives or it doesn't — the application has to handle that if it cares.
- Much lower overhead and latency per packet as a direct consequence of skipping all of the above.

| | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Guarantees | Reliable, ordered, retransmits lost data | None |
| Overhead | Higher (headers, acks, handshake) | Minimal |
| Used for | HTTP, databases, file transfer, banking | Video streaming, gaming, VoIP, DNS queries |

**Rule of thumb for interviews:** if correctness/completeness matters more than raw speed → TCP. If freshness matters more than a dropped packet or two (a late/lost video frame is worthless anyway, so why retransmit it) → UDP.

**Sockets & ports:** a socket is the OS-level endpoint — an **(IP address, port) pair** — that a program binds to. Ports let multiple applications on one machine each have their own channel: ports 0–1023 are "well-known" (80 = HTTP, 443 = HTTPS, 22 = SSH, 25 = SMTP), 1024–49151 are "registered" for specific applications, and above that are typically used as ephemeral/dynamic ports a client picks temporarily for an outgoing connection.

**NAT (Network Address Translation) — worth understanding here since it resurfaces later (WebRTC, FTP):** most devices on a home/office network share one public IP. The router rewrites outgoing packets to use its own public IP + a chosen port, and remembers the mapping so return traffic gets routed back to the right internal device. This is exactly why a server can't casually open a new connection back to a client sitting behind NAT — the router has no mapping for an inbound connection it didn't initiate.

---

## 1.2 HTTP/HTTPS deep dive

**Methods** (and whether repeating the call is safe = idempotent):

| Method | Purpose | Idempotent? |
|---|---|---|
| GET | Read a resource | Yes |
| POST | Create a resource | No |
| PUT | Replace a resource entirely | Yes |
| PATCH | Partially update a resource | No (usually) |
| DELETE | Remove a resource | Yes |

Idempotency matters because in distributed systems requests get retried after timeouts — if a client retries a POST because it never saw the response, did it just create the resource twice? (Direct link forward to 5.12 Idempotency in the roadmap — this is where that problem originates.)

**Status codes** — know these cold:
- `2xx` success: `200 OK`, `201 Created`, `204 No Content`
- `3xx` redirect: `301 Moved Permanently`, `304 Not Modified` (tells the client its cached copy is still valid — saves bandwidth)
- `4xx` client error: `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `429 Too Many Requests`
- `5xx` server error: `500 Internal Server Error`, `502 Bad Gateway` (upstream server gave an invalid response), `503 Service Unavailable`

**Headers that actually come up in design discussions:**
- `Content-Type` — what format the body is (`application/json`, `multipart/form-data` for file uploads).
- `Authorization` — credentials (e.g., `Bearer <JWT>`).
- `Cache-Control` — caching rules (`max-age=3600`, `no-cache`, `private` vs `public`) — directly drives CDN and browser caching behavior.
- `ETag` — a hash/version identifier for a resource; the client sends it back via `If-None-Match` on the next request, and the server replies `304 Not Modified` if nothing changed, avoiding re-sending the whole body.
- `Transfer-Encoding: chunked` — lets the server start streaming a response before it knows the total length (as opposed to `Content-Length`, which requires knowing the full size upfront).
- Cookies — the mechanism that adds state on top of an otherwise stateless protocol (see below).

**HTTP is stateless by design** — each request is independent, the server retains no memory of previous requests from the same client. Cookies (a header set by the server, echoed back by the client on every subsequent request) are the bolt-on mechanism used to simulate state (sessions, login) on top of a stateless protocol.

**HTTPS = HTTP + TLS.** The TLS handshake (simplified):
1. Client sends "hello" — supported TLS versions and cipher suites.
2. Server responds with its **certificate** (which a trusted **Certificate Authority** has signed, vouching "this public key really belongs to this domain") and picks a cipher suite.
3. Client verifies the certificate against its trusted CA list, then client and server perform a key exchange (commonly **Diffie-Hellman**) to agree on a **shared symmetric session key** — without ever transmitting that key itself over the network.
4. All further traffic is encrypted with that fast symmetric key (asymmetric crypto is used only briefly, for the handshake, because it's computationally expensive — symmetric crypto handles the actual bulk data efficiently).

**HTTP/1.1 vs HTTP/2 vs HTTP/3:**
- **HTTP/1.1** — text-based, one request waits for its response before the next can be sent on that connection (browsers work around this by opening multiple parallel TCP connections, which has its own overhead).
- **HTTP/2** — binary framing, **multiplexes many requests over a single TCP connection** concurrently (no more waiting in line), plus header compression (HPACK) since headers repeat constantly across requests. Downside: because it's still riding on one TCP connection, a single lost packet stalls *all* multiplexed streams until it's retransmitted (TCP head-of-line blocking still exists at the transport layer, just not the application layer).
- **HTTP/3** — runs over **QUIC**, which is built on **UDP** instead of TCP, and re-implements reliability itself at the application layer per-stream. This removes head-of-line blocking entirely (one lost packet only stalls the one stream it belongs to), and supports faster connection setup (0-RTT resumption for repeat connections) and **connection migration** (a connection survives switching from WiFi to cellular, since QUIC identifies connections by an ID rather than the IP/port tuple TCP relies on).

---

## 1.3 DNS: how resolution works

DNS is a distributed, hierarchical, heavily-cached lookup system that turns `example.com` into an IP address.

**Record types worth knowing:**
- `A` — hostname → IPv4 address
- `AAAA` — hostname → IPv6 address
- `CNAME` — alias, points one hostname to another hostname (not directly to an IP)
- `MX` — which mail server handles email for this domain (used by SMTP, see 1.7)
- `NS` — which name servers are authoritative for this domain
- `TXT` — arbitrary text, commonly used for domain ownership verification and email anti-spam records (SPF/DKIM)

**Resolution flow, in order:**
1. Browser cache (has this been looked up recently in this browser session?)
2. OS-level cache
3. **Recursive resolver** — usually your ISP's, or a public one like `8.8.8.8`/`1.1.1.1`. This is the server that does the actual legwork on your behalf.
4. The resolver queries a **root server** — this doesn't know the final answer, but knows which server handles the top-level domain (`.com`, `.org`, etc.).
5. The resolver queries the **TLD server** for `.com` — this doesn't know the final IP either, but knows which name server is *authoritative* for `example.com` specifically.
6. The resolver queries the **authoritative name server** for `example.com`, which finally returns the actual IP.
7. The answer is cached at every layer along the way (browser, OS, resolver) for the duration of the record's **TTL**.

This is technically a mix of **recursive** queries (your resolver → the world, doing all the work) and **iterative** queries (root → TLD → authoritative, each just pointing further, never doing the work themselves) — your device only ever does one recursive query; the resolver does the iterative chasing.

**TTL (time-to-live):** how long a record can be cached before it must be re-checked. Low TTL = changes propagate fast but more DNS query load and slightly higher average latency; high TTL = opposite. This is why teams *lower* the TTL on a DNS record *before* a planned migration — so that when they actually flip the record, clients pick up the change quickly instead of some fraction of users being stuck on stale (potentially now-dead) IPs for hours.

**Anycast:** the same IP address is announced from multiple physical locations via BGP (the protocol routers use to exchange "who can reach what" information), and normal internet routing sends each client to whichever announcing location is topologically nearest. This is how CDNs and the DNS root servers themselves achieve low latency globally without the client doing anything special — it's invisible load balancing at the network-routing layer, not the application layer.

**GeoDNS / DNS-based load balancing:** a DNS server can also return *different* IPs to different clients based on the resolver's location or based on health checks of backend servers — this is a coarse-grained way to route traffic to the nearest datacenter or away from a failed one, layered on top of (or instead of) anycast.

---

## 1.4 Vertical vs horizontal scaling

- **Vertical (scale up):** add more CPU/RAM/disk to one existing machine. Simple — no architecture changes required — but hits a hard hardware ceiling, remains a single point of failure the whole time, and typically needs downtime (or at least a failover) to actually resize.
- **Horizontal (scale out):** add more machines behind a load balancer. No hard ceiling, and you get redundancy essentially for free — but it requires real architectural work:
  - The application must become **stateless** — if a server keeps session data in its own local memory, a request that lands on a *different* server next time won't have that context. This is the **sticky session problem**: either force a client's requests back to the same server every time (fragile — that server becomes a soft single point of failure for that user) or push session state out to a shared store (e.g., Redis) that any server can read.
  - The **database** usually can't just be horizontally scaled the same trivial way the stateless app servers can — this is the entire subject of Phase 3 (replication, partitioning). A stateless web tier scaling out is the easy 20%; the data tier is the hard 80%.
  - **Auto-scaling groups** — cloud infrastructure that watches a metric (CPU, request queue depth) and automatically adds/removes server instances — is the practical mechanism that makes horizontal scaling operationally manageable instead of manual.

**Interview takeaway:** almost every "design X at scale" problem assumes horizontal scaling of the application tier is a given — the actual interesting design work is in *what has to change* to make the data layer scale horizontally too.

---

## 1.5 Latency numbers every engineer should know

Rough orders of magnitude — memorize the *relative gaps* between rows, not exact numbers (these shift with hardware generations, but the ratios barely do):

| Operation | Approx. latency |
|---|---|
| L1 cache reference | ~1 ns |
| Mutex lock/unlock | ~20-25 ns |
| RAM access | ~100 ns |
| Compress small payload in memory | ~1-10 µs |
| SSD random read | ~100 µs (0.1 ms) |
| Round trip within same datacenter | ~0.5 ms |
| Disk seek (spinning disk) | ~10 ms |
| Round trip cross-country / cross-region | ~50–150 ms |

**Why this matters in interviews:** it's the entire justification for caching, and for keeping chatty calls within one datacenter/region. "Why add Redis in front of the database?" → RAM access is roughly **1,000x faster than an SSD read**, and a same-datacenter network round trip is **thousands of times slower than RAM** — that gap is what a cache hit is buying you. Similarly, "why does this design put a read replica in each region instead of one global database?" → because a single cross-region round trip alone can cost more time than dozens of local operations combined.

---

## 1.6 Concurrency basics

- **Process:** an isolated unit with its own memory space. Heavier to create/switch between, but a crash in one process doesn't corrupt another.
- **Thread:** lives inside a process and **shares its memory** with other threads in that process. Lighter weight to create and switch between than a process, but a bug in one thread (a race condition, a bad write) can corrupt state shared with every other thread in that process.
- **Race conditions & locks (the reason threading is hard):** if two threads read-modify-write the same piece of memory without coordination, the final result depends on unpredictable timing. A **mutex/lock** forces only one thread at a time into that critical section — correct, but a source of contention and potential deadlock if locks are acquired in inconsistent orders.
- **Blocking vs non-blocking I/O:** a blocking call (e.g. a network read) halts the calling thread until the operation completes. A non-blocking call returns immediately whether or not data is ready, leaving it to the caller to check back.
- **The event loop / async I/O model** (Node.js, and under the hood most modern async frameworks): a small number of threads (often just one) handles many concurrent connections by *never blocking* — instead of waiting on I/O, it registers interest with the OS (via a mechanism like `epoll` on Linux or `kqueue` on BSD/macOS) and moves on to other work, getting notified later when data is actually ready. This is how one process can hold thousands of idle connections cheaply — there's no per-connection thread sitting around consuming stack memory while waiting.
- **Thread-per-request model** (traditional Java/Tomcat-style servers): simple mental model — each incoming request gets its own thread for its entire lifetime — but each thread has real memory overhead (stack space) and OS scheduling cost, so this approach runs out of headroom well before an event-loop model does under high concurrent connection counts. This specific failure mode — a server falling over trying to hold ~10,000 concurrent connections — is historically known as the **C10K problem**, and is the direct motivation behind event-loop architectures.
- **Middle ground worth knowing exists:** languages like Go (goroutines) and newer async runtimes implement lightweight "green threads" scheduled by the language runtime rather than the OS — cheaper than OS threads, without forcing the callback-heavy style that pure event-loop code can turn into.

**Why this matters for system design:** when an interviewer asks "how many concurrent connections can this one server handle," the honest answer depends entirely on which of these models it uses — this is a real, concrete axis of comparison, not a detail to hand-wave past.

---

## 1.7 Protocol zoo — real-time delivery, messaging, email, P2P, RPC, file/remote access

### Real-time delivery to a browser/client

| Approach | How it works | Tradeoff |
|---|---|---|
| Long-polling | Client asks, server holds the request open until there's data (or a timeout), then the client immediately re-asks | Works everywhere HTTP works, but overhead of repeated request setup |
| WebSockets | A single persistent, full-duplex TCP connection, upgraded from an initial HTTP handshake | Low latency, low overhead, but the server must hold open connection state per client |
| SSE (Server-Sent Events) | Server pushes a one-way stream of events to the client over a single long-lived HTTP connection | Simpler than WebSockets (plain HTTP, auto-reconnect built into the spec), but strictly one-directional (server → client only) |

### AMQP (Advanced Message Queuing Protocol)

**The problem it solves:** before AMQP, every message broker (IBM MQ, TIBCO, etc.) had its own proprietary wire format — a producer built for one vendor's broker couldn't talk to another vendor's broker. AMQP's entire point is being a **wire-level protocol**: it standardizes the actual bytes sent over the network, not just an API. (Contrast with Java's JMS, which only standardizes an API — a JMS client is still locked to one vendor's broker implementation underneath.)

**The core model — Producer → Exchange → Queue → Consumer:** producers never publish directly to a queue. They publish to an **exchange**, and the exchange decides which queue(s) the message lands in based on **bindings** and a **routing key**. This indirection is the whole design.

**Exchange types:**
- **Direct** — routes to the queue(s) whose binding key exactly matches the message's routing key.
- **Fanout** — ignores the routing key, broadcasts to every bound queue. This is how you implement broadcast pub/sub (e.g., "notify all services when an order is placed").
- **Topic** — pattern matching on the routing key with wildcards: `*` matches exactly one word, `#` matches zero or more. A queue bound to `"order.#"` catches `"order.us.created"` and `"order.us.eu.created"` alike; bound to `"order.*"` only catches the single-word case.
- **Headers** — routes based on message header key/value pairs instead of a routing key string (rarely used).

**Delivery guarantees & acknowledgment:** a consumer explicitly **acks** after successfully processing a message; until then, the broker keeps it. If a consumer disconnects or **nacks**, the broker **redelivers** — meaning you get **at-least-once delivery** by default, so consumer logic must be idempotent (the same message can legitimately arrive twice). **Durability** flags on queues/messages mean they're written to disk and survive a broker restart; without it, they're in-memory only and lost on crash.

**Tools:** RabbitMQ (by far the most common AMQP implementation), Apache Qpid, Azure Service Bus (partial AMQP support). Note AMQP 0-9-1 (what RabbitMQ primarily speaks) and AMQP 1.0 (a more general OASIS standard) are meaningfully different specs.

**Critical distinction — RabbitMQ/AMQP vs Kafka (these get conflated constantly):**
- **RabbitMQ:** queue-based — the broker tracks per-message delivery state and **removes the message once acked**. Good fit for task queues and complex routing.
- **Kafka:** log-based — messages are appended to a durable, partitioned, ordered log; the broker doesn't track delivery per consumer — **consumers track their own offset** and can replay from any point, and many independent consumer groups can read the same data at their own pace. Good fit for event streaming and "replay history" requirements.

If a design problem needs replay or multiple independent readers of the same stream, that's Kafka's model, not RabbitMQ's.

### MQTT (Message Queuing Telemetry Transport)

**Origin:** designed in 1999 to monitor oil pipelines over **satellite links** — extremely low bandwidth, high latency, unreliable. Every design decision traces back to minimizing bytes on the wire.

**Model:** publish/subscribe via a broker. Publishers send to a **topic** string (e.g. `sensors/livingroom/temperature`); subscribers use wildcards — `+` matches exactly one topic level, `#` matches everything below that point.

**QoS levels (the core mechanic):**
- **QoS 0 — at most once:** fire and forget, no ack, fastest, message can silently vanish.
- **QoS 1 — at least once:** `PUBLISH` → `PUBACK`; if the ack is lost the publisher resends, so the subscriber might see a duplicate.
- **QoS 2 — exactly once:** a 4-step handshake (`PUBLISH` → `PUBREC` → `PUBREL` → `PUBCOMP`) guaranteeing exactly-once delivery, at the cost of more round trips — used when duplicates would cause real harm (e.g., "unlock the door").

**Other mechanics:**
- **Retained messages** — the broker keeps the *last* message on a topic, so a client subscribing later immediately gets the last known value instead of waiting for the next update.
- **Last Will and Testament (LWT)** — a client registers a message to be auto-published *if it disconnects ungracefully* — how MQTT systems detect "device went offline" without polling.
- **Persistent sessions** — the broker remembers a client's subscriptions and queues messages while it's briefly disconnected.
- Tiny header overhead (as low as 2 bytes) because constrained IoT hardware can't afford verbose protocols like HTTP.

**Tools:** Mosquitto (lightweight, most common open-source broker), HiveMQ, EMQX, AWS IoT Core, Azure IoT Hub.

### Email: SMTP, IMAP, POP3 — three protocols for three separate jobs

**SMTP (Simple Mail Transfer Protocol)** — a **push/relay** protocol only, no concept of "read my inbox." Two uses: **submission** (your client sending to your provider, port 587 + STARTTLS + auth) and **relay** (server-to-server, port 25). **Store-and-forward:** a message hops server to server, each hop looking up the destination domain's **MX record** in DNS to find the accepting server. Command sequence: `EHLO` → `MAIL FROM` → `RCPT TO` → `DATA` → `.` → `QUIT`.

**IMAP (Internet Message Access Protocol)** — **server-centric**: mail and its state (read/unread, folders, flags) live on the server; clients are just a window into it — which is exactly why deleting an email on your phone deletes it everywhere. The **IDLE** command lets a client hold a connection open and get pushed a notification the instant new mail arrives, instead of polling. Supports **partial fetch** — grab just headers or one MIME part, skipping a large attachment on a slow connection.

**POP3 (Post Office Protocol v3)** — much simpler and older: connect, `USER`/`PASS`, `LIST`, `RETR`, `DELE`, `QUIT`. Classic default behavior **downloads mail to one device and deletes it from the server** — a second device never sees that message (most clients now configure "leave a copy on server" as a workaround, but that's client-side, not protocol-native). No folder concept, no cross-device sync — exactly why IMAP displaced it once people used multiple devices.

### WebRTC (Web Real-Time Communication)

**The goal:** let two browsers exchange audio/video/data **directly**, peer-to-peer, without routing media through a central server. The catch: WebRTC deliberately does **not** define how the two peers first find each other — that's left entirely to you.

**The pieces you have to assemble:**
1. **Signaling** (not part of the spec) — before connecting, peers must exchange **SDP** (session description — codecs, resolution, etc.) and **ICE candidates**. This exchange needs *some* server — typically a WebSocket server relaying this one-time handshake, never the actual media.
2. **ICE (Interactive Connectivity Establishment)** — gathers candidate addresses (local IP, public IP via STUN, relay via TURN) and tries them in order to find the best path given both peers are likely behind NATs.
3. **STUN** — a lightweight server that tells a client "here's your public IP:port as seen from outside," so two NATed peers can learn each other's real reachable address.
4. **TURN** — the fallback relay server used when a direct connection genuinely isn't possible (e.g., symmetric NAT on both sides). All media routes through it — no longer truly P2P at that point, just transparent to the app. TURN capacity is the expensive part of running a video product at scale, since you're paying for relayed media bandwidth.
5. **DTLS-SRTP** — encrypts the media stream once connected.

**Data channels:** `RTCDataChannel` sends arbitrary (non-media) data peer-to-peer too — used for P2P file transfer or real-time game state.

**Design-interview relevance:** for a "design a video calling app" question, the honest answer is media *can* be P2P via WebRTC for 1:1 calls, but group calls beyond a handful of participants typically route through an **SFU (Selective Forwarding Unit)** media server instead of pure P2P mesh, since mesh bandwidth cost grows with the square of participant count — plus you still need to provision signaling and TURN infrastructure regardless.

### RPC (Remote Procedure Call)

Not one protocol — a **paradigm**: calling a function that executes on a different machine, made to look like a local call. Every RPC system needs a **client stub** (serializes/"marshals" the arguments, sends them) and a **server stub** (deserializes, invokes the real function, serializes the result back) — this marshalling need is exactly why an IDL (Interface Definition Language) exists, generating matching stubs across languages from one shared contract.

**Landscape:** Sun/ONC RPC (1980s, powers NFS) → CORBA (1990s, notoriously complex, mostly historical) → Java RMI (Java-only) → modern default **gRPC** (Google): runs over **HTTP/2**, uses **Protocol Buffers** for compact binary serialization, strongly typed via `.proto` files with code-gen for many languages, and supports unary, server-streaming, client-streaming, and full bidirectional-streaming calls — the streaming support is a major reason it's popular for internal service-to-service traffic where REST's request/response shape is awkward. **Thrift** (originally Facebook) is conceptually similar. **JSON-RPC/XML-RPC** are much simpler and human-debuggable but lower performance.

**The semantic trap every RPC system faces:** if a call times out, the client **cannot tell** whether the server never received it, processed it and crashed before responding, or processed it and the response itself was lost. This is why retried RPC calls need idempotent design (same underlying issue as 5.12 in the roadmap) — at-least-once delivery is "free," exactly-once requires deliberate work (e.g., idempotency keys).

**RPC vs REST, in one line:** RPC is **action-oriented** (`chargeCard()`), REST is **resource-oriented** (`POST /charges`). RPC tends to win for internal service-to-service calls (performance, strong typing, streaming); REST tends to win for public APIs (cacheability, discoverability, broad compatibility).

### FTP vs SSH/SFTP — commonly confused, architecturally unrelated

**FTP** uses **two separate TCP connections** — a control connection (port 21, commands) and a separate data connection for the actual file bytes. **Active mode** has the *server* initiate the data connection back to the client, which routinely breaks behind NAT/firewalls; **passive mode (PASV)** has the *client* initiate both connections instead, which is why it's the practical default. FTP is **plaintext by default** — a real security problem. **FTPS** = FTP with TLS layered on top, still using the two-connection model.

**SFTP is not FTP at all**, despite the name — it's a subsystem that runs entirely **over SSH**, using SSH's single existing encrypted connection and its own binary packet protocol for file operations. One connection, always encrypted, no active/passive confusion — the practical default recommendation today over both FTP and FTPS.

**SSH** is layered: (1) **transport layer** — authenticates the server, negotiates encryption via key exchange (e.g. Diffie-Hellman) to get a shared symmetric key, then encrypts traffic (e.g. AES); (2) **user authentication layer** — password auth (phishable) or **public key auth**, where the client proves possession of a private key cryptographically and the key/password itself is *never* transmitted, which is why it's preferred for production servers; (3) **connection layer** — multiplexes multiple logical channels (a shell session and a port-forward simultaneously) over the one encrypted tunnel. **Port forwarding** — local (tunnel a local port to a remote destination), remote (the reverse), and dynamic (turns SSH into a SOCKS proxy) — lets arbitrary TCP traffic ride through the encrypted connection.

---

## Comprehension checks

**1.** You're designing an app where a mobile client needs live GPS location updates from a delivery driver every 2 seconds, and separately needs to upload a delivery confirmation photo. Which transport/protocol would you pick for each, and why?

<details>
<summary>Expand</summary>

- **Live GPS updates:** WebSockets (or UDP if you control both ends) — frequent, low-latency, small payloads where an occasional dropped update doesn't matter. A persistent connection avoids repeated setup overhead.
- **Photo upload:** HTTPS POST over TCP — needs reliability (the photo must arrive complete and uncorrupted), and it's a one-off request rather than a continuous stream, so TCP's guarantees are worth their overhead here.

</details>

**2.** A smart-home system has thousands of battery-powered sensors reporting temperature every 30 seconds over spotty WiFi, and a mobile app that needs to see the *current* temperature immediately on open, even if no reading has fired in the last few minutes. Which protocol fits the sensor-to-cloud reporting, and which specific feature solves "show current value instantly"?

<details>
<summary>Expand</summary>

- **Sensor → cloud:** MQTT — built for low-power, unreliable-network, many-small-device scenarios. QoS 1 is reasonable (an occasional duplicate reading is harmless; QoS 0 risks silently losing readings).
- **"Show current value instantly":** MQTT's **retained message** feature — the broker holds the last published value per topic, so the moment the app subscribes, it gets the last known reading immediately without waiting for the next cycle.

</details>
