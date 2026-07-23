# Phase 0 — Interview Framework

This is the lens you use for every system design problem going forward — before touching any specific technology.

---

## 0.1 What the interview actually tests

There is no "correct" architecture for "design Twitter." The interviewer is watching **how you think**, not whether you recite the right diagram. Specifically they're evaluating:

- **Handling ambiguity** — real requirements are never given upfront; do you ask, or do you guess?
- **Structured thinking** — do you have a repeatable process, or do you flail?
- **Tradeoff reasoning** — every decision (SQL vs NoSQL, sync vs async) has a cost. Can you name it?
- **Communication** — can you narrate your thinking so a teammate could follow along?
- **Adaptability** — if the interviewer says "now we have 100x the users," can you adjust the design instead of restarting from scratch?

**Biggest red flag:** jumping straight to "I'll use Kafka and Redis" with no justification. Naming technology isn't the skill — justifying *why that tool solves this constraint* is.

---

## 0.2 The standard framework (use this every time, no exceptions)

A 45-minute interview roughly breaks down like this:

| Step | Time | What happens |
|---|---|---|
| 1. Requirements | ~5 min | Clarify what's in/out of scope |
| 2. Estimation | ~5 min | Back-of-envelope scale numbers |
| 3. API design | ~5 min | Define the contract (endpoints/inputs/outputs) |
| 4. High-level design | ~10-15 min | Boxes and arrows — the big picture |
| 5. Deep dive | ~10-15 min | Interviewer picks 1-2 components to go deep on |
| 6. Bottlenecks/scale | remaining | Where does this break, and how do you fix it |

Skipping step 1 (requirements) is the #1 way candidates fail — they design a system for a problem the interviewer never asked for.

---

## 0.3 Functional vs non-functional requirements

- **Functional** = what the system *does*. E.g., "users can post a tweet," "users can follow other users."
- **Non-functional** = the *qualities* the system must have. E.g., availability, latency, consistency, durability, scalability.

Interviews are won or lost on non-functional requirements, because they dictate every downstream decision. Questions to always ask:
- How many users, and what's the read:write ratio? (Most systems are read-heavy — this changes your caching strategy.)
- Does this need strong consistency, or is eventual consistency fine? (A bank balance ≠ a "like" count.)
- What's the acceptable latency? (Real-time chat vs a nightly report have wildly different designs.)

---

## 0.4 Back-of-envelope estimation

You need to move fast here — this isn't precision math, it's sanity-checking scale. Core formulas:

- **QPS (queries/sec)** ≈ (daily active users × actions per user per day) / 86,400
- **Storage** ≈ records × size per record (then project 1yr/5yr growth)
- **Bandwidth** ≈ QPS × average payload size

**Worked example:** Twitter-scale, 200M daily active users, each posts 2 tweets/day, each tweet ~300 bytes:
- Writes/day = 400M tweets → QPS ≈ 400M / 86,400 ≈ **~4,600 writes/sec** (average; peak could be 3-5x that)
- Storage/day ≈ 400M × 300 bytes ≈ **120 GB/day** just for tweet text

This kind of number tells you immediately: "this needs horizontal partitioning, a single Postgres box won't survive this."

---

## 0.5 How Microsoft interviews differ

- Style tends to be **collaborative, not adversarial** — the interviewer often nudges ("what if traffic 10x'd?") rather than staying silent. Treat those nudges as gifts, not traps.
- No requirement to know Azure specifically, but being comfortable with the *concepts* behind services like Blob Storage, Cosmos DB, or Service Bus helps you reason faster.
- The system design round is sometimes blended with a light behavioral question at the start ("tell me about a project where you made a tradeoff") — don't be thrown if that happens.
- They weight **communication and structure** heavily since the role requires working with distributed teams.

---

## Comprehension check
Given "design a system where 50M users check a live sports score that updates every 5 seconds," what's the first non-functional requirement question you'd ask?

<details>
<summary>Think it through, then expand</summary>

Good first questions:
- Does this need strong consistency (everyone sees the exact same score at the exact same moment) or is a few seconds of staleness acceptable?
- Is this read-heavy (millions watching) vs write-heavy (score updates are rare, from one source)?
- Real-time push (WebSocket/SSE) vs polling — what latency is "acceptable"?

</details>
