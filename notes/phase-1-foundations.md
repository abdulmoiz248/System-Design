# Phase 1 — Foundations

Everything in system design sits on top of these mechanics. You don't need to become a networking engineer, but you need enough intuition to answer "why is this slow" or "why did this fail" without hand-waving.

---

## 1.1 How computers talk: client-server, IP, TCP vs UDP, sockets

**Client-server model:** a client initiates a request, a server responds. Almost every system design problem is really "how do we scale this basic pattern to millions of clients."

**IP (Internet Protocol):** every machine has an address (IPv4 like `192.168.1.1`, or IPv6 for the larger modern address space). Data travels as **packets**, routed hop-by-hop across routers — IP itself makes no promise the packet arrives, or arrives in order.

**TCP vs UDP** — the two transport-layer protocols built on top of IP:

| | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (3-way handshake: SYN → SYN-ACK → ACK) | Connectionless — just sends |
| Guarantees | Reliable, ordered delivery, retransmits lost packets | None — packets can drop or arrive out of order |
| Speed | Slower (overhead of guarantees) | Faster |
| Used for | HTTP, databases, file transfer, banking | Video streaming, gaming, VoIP, DNS queries |

**Rule of thumb for interviews:** if correctness/completeness matters more than speed → TCP. If speed/freshness matters more than a dropped packet or two → UDP (a lost video frame doesn't matter; a lost bank transaction does).

**Sockets:** the OS-level endpoint — an (IP address, port) pair — that a program binds to, so multiple applications on one machine can each have their own communication channel.

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

Idempotency matters a lot in distributed systems — if a client retries a POST after a timeout, did it create the resource twice? (This connects to 5.12 Idempotency in the roadmap.)

**Status codes** — know these cold:
- `2xx` success: `200 OK`, `201 Created`, `204 No Content`
- `3xx` redirect: `301 Moved Permanently`, `304 Not Modified`
- `4xx` client error: `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `429 Too Many Requests`
- `5xx` server error: `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`

**Headers:** metadata riding alongside the request/response — e.g. `Content-Type` (what format the body is), `Authorization` (credentials), `Cache-Control` (caching rules).

**HTTPS = HTTP + TLS.** The TLS handshake (simplified):
1. Client says hello (supported ciphers)
2. Server responds with its certificate (proves identity) + chosen cipher
3. Client and server agree on a shared **symmetric session key** via asymmetric key exchange
4. All further traffic is encrypted with that fast symmetric key

**HTTP/1.1 vs HTTP/2 vs HTTP/3:**
- HTTP/1.1: one request per connection at a time (or limited pipelining) — head-of-line blocking
- HTTP/2: multiplexes many requests over a single TCP connection, header compression — much faster
- HTTP/3: runs over **QUIC** (built on UDP instead of TCP) — removes TCP-level head-of-line blocking entirely, faster connection setup

---

## 1.3 DNS: how resolution works

DNS is a distributed, hierarchical lookup system that turns `example.com` into an IP address.

**Resolution flow:** browser cache → OS cache → recursive resolver (usually your ISP or 8.8.8.8) → root server → TLD server (`.com`) → authoritative name server → IP returned and cached at every layer along the way.

**TTL (time-to-live):** how long a DNS record can be cached before it must be re-checked. Low TTL = faster propagation of changes but more DNS query load; high TTL = opposite.

**Anycast:** the same IP address is announced from multiple physical locations, and network routing sends each client to the *nearest* one. This is how CDNs and DNS root servers achieve low latency globally without the client needing to know anything about geography.

---

## 1.4 Vertical vs horizontal scaling

- **Vertical (scale up):** add more CPU/RAM/disk to one machine. Simple, no architecture changes needed — but hits a hardware ceiling, remains a single point of failure, and often needs downtime to resize.
- **Horizontal (scale out):** add more machines. No hard ceiling, and gives you redundancy — but requires the application to be stateless (or push state to a shared store), needs a load balancer, and introduces every distributed-systems problem in Phase 3 (replication, partitioning, consistency).

**Interview takeaway:** almost every "design X at scale" problem assumes horizontal scaling is inevitable — your job is to explain *what has to change* in the design to make that possible (statelessness, sharding, etc.).

---

## 1.5 Latency numbers every engineer should know

Rough orders of magnitude (memorize the *relative* gaps, not exact numbers):

| Operation | Approx. latency |
|---|---|
| L1 cache reference | ~1 ns |
| RAM access | ~100 ns |
| SSD random read | ~100 µs (0.1 ms) |
| Round trip within same datacenter | ~0.5 ms |
| Disk seek (spinning disk) | ~10 ms |
| Round trip cross-country / cross-region | ~50–150 ms |

**Why this matters in interviews:** it's the entire justification for caching. "Why add Redis?" → because a cache hit is ~100x-1000x faster than a database round trip, and that difference compounds at scale.

---

## 1.6 Concurrency basics

- **Process:** has its own isolated memory space. Heavier to create, but crashes don't take down other processes.
- **Thread:** lives inside a process, shares its memory. Lighter weight, but a bug in one thread can corrupt shared state or crash the whole process.
- **Blocking vs non-blocking I/O:** a blocking call halts the thread until the operation (e.g. a DB query) finishes. Non-blocking lets the thread do other work while waiting.
- **Async I/O / event loop model** (e.g. Node.js): a single thread handles many concurrent connections by never blocking — it registers a callback and moves on. Contrast with **thread-per-request** (e.g. traditional Java/Tomcat servers): simple mental model, but you run out of threads under high concurrency (the "C10K problem" — handling 10,000 concurrent connections).

**Why this matters:** it's why a single Node.js server can hold open thousands of idle WebSocket connections cheaply, while a thread-per-request model would fall over from thread overhead alone.

---

## 1.7 Protocol zoo (beyond HTTP)

Grouped by what problem each one solves:

**Real-time delivery to a browser/client:**
| Approach | How it works | Tradeoff |
|---|---|---|
| Long-polling | Client asks, server holds the request open until there's data | Simple, but overhead per request |
| WebSockets | Persistent, full-duplex connection | Low latency, low overhead, more server-side connection state to manage |
| SSE (Server-Sent Events) | Server pushes a one-way stream to client over HTTP | Simpler than WebSockets, but one-directional only |

**Messaging between services:**
- **AMQP** — protocol behind message brokers like RabbitMQ; supports queuing, routing, delivery guarantees.
- **MQTT** — lightweight publish/subscribe protocol built for low-power IoT devices.

**Email:**
- **SMTP** — sending mail (client→server or server→server)
- **IMAP** — reading mail; state lives on the server (delete on phone → deleted everywhere)
- **POP3** — older; downloads mail to one device and removes it from the server

**Other:**
- **WebRTC** — peer-to-peer real-time media (video calls) without routing through a central server
- **RPC** — a function call that actually executes on a different machine/process, made to look local
- **FTP vs SSH** — FTP transfers files (insecure by default); SSH is an encrypted tunnel for remote login/administration (SFTP rides on top of SSH for secure file transfer)

---

## Comprehension check

You're designing an app where a mobile client needs live GPS location updates from a delivery driver every 2 seconds, and separately needs to upload a delivery confirmation photo. Which transport/protocol would you pick for each, and why?

<details>
<summary>Think it through, then expand</summary>

- **Live GPS updates:** WebSockets (or plain UDP if you control both ends) — frequent, low-latency, small payloads where an occasional dropped update doesn't matter much. A persistent connection avoids the overhead of repeated polling.
- **Photo upload:** plain HTTPS POST over TCP — you need reliability (the photo must arrive complete and uncorrupted), and it's a one-off request, not a continuous stream, so the overhead of TCP's guarantees is worth it here.

</details>
