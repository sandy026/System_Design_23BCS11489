# Trade-offs — Real-Time Chat System

---

## 1. WebSockets vs. Long Polling vs. Server-Sent Events (SSE)

### Options Considered

| Mechanism | Description |
|---|---|
| **WebSockets** | Full-duplex, persistent TCP connection; both client and server can push data at any time |
| **Long Polling** | Client sends HTTP request; server holds it open until data is available, then closes it |
| **Server-Sent Events (SSE)** | One-directional, server → client push over HTTP/2 |

### Decision: **WebSockets**

**Why:**
- Chat is **bidirectional** by nature (typing indicators, read receipts, message delivery — all flow in both directions simultaneously). WebSockets support this natively.
- Once established, a WebSocket connection has **no per-message HTTP overhead**, unlike polling which adds headers on every request.
- **Latency**: WebSockets achieve sub-50 ms delivery in practice; Long Polling can add 100–500 ms depending on poll interval.

**Trade-offs Accepted:**
- WebSockets are **stateful** — load balancers need sticky sessions or a shared pub/sub layer (Redis) to route messages to the correct server.
- WebSockets are **harder to scale** than stateless REST APIs; we mitigate this with a Layer-4 NLB and Redis pub/sub.
- Some **corporate firewalls and proxies** block WebSocket upgrades → we implement HTTP Long Polling as a graceful fallback.
- **Mobile battery**: Keeping a TCP connection alive drains battery → we use a 30-second heartbeat and allow the OS to coalesce connections where possible.

---

## 2. SQL (PostgreSQL) vs. NoSQL (Cassandra) for Messages

### Decision: **Cassandra for messages; PostgreSQL for user/group metadata**

### Messages → Cassandra

**Why:**
- Messages are **write-heavy** (100 K writes/second at peak); Cassandra is designed for exactly this workload.
- **Access pattern is simple and predictable**: "Give me the last N messages in conversation X, newest first" — maps perfectly to Cassandra's partition + clustering key model.
- Cassandra **scales horizontally without resharding** — just add nodes.
- **No joins needed** for message retrieval; the document-style row format is sufficient.
- Built-in **multi-datacenter replication** with tunable consistency.

**Trade-offs Accepted:**
- Cassandra does **not support complex queries** (no JOINs, no aggregations, no ad-hoc filtering). All queries must be pre-planned around partition keys.
- **No ACID transactions** across partitions — acceptable because message ordering within a conversation is sufficient.
- **Data modelling requires upfront thought**: changing the schema later is expensive.

### User/Group Metadata → PostgreSQL

**Why:**
- User, group, and membership data involves **relational queries** (e.g., "find all groups a user is in", "check if two users are blocked").
- **ACID transactions** are critical — we never want a user created without their presence record, or a group member added without the membership row.
- Volume is much lower (millions of records, not billions).

---

## 3. Eventual Consistency vs. Strong Consistency

### Decision: **Eventual consistency for messages; strong consistency for user accounts**

**Messages (Cassandra — `LOCAL_QUORUM` write, `LOCAL_ONE` read):**
- We tolerate a **brief window of inconsistency** (< 100 ms typically) where one node might not yet have the latest message.
- In practice, **read-repair** and Cassandra's hinted handoff bring nodes into sync quickly.
- Trade-off: In rare cases, a user might briefly see an out-of-order message on reconnect → the client re-sorts by `sent_at` timestamp.

**User Accounts (PostgreSQL — synchronous replication to at least 1 replica):**
- Strong consistency is required here: if a user changes their password or blocks someone, that change must be immediately visible to all servers.

---

## 4. Message Fan-Out Strategy: Push vs. Pull

### Options

| Approach | How it works |
|---|---|
| **Push (fan-out on write)** | When a message is sent, immediately push it to all recipients' devices via WebSocket / push notification |
| **Pull (fan-out on read)** | Store the message once; when a user opens a conversation, they pull the latest messages |
| **Hybrid** | Push to online users via WebSocket; offline users pull when they reconnect |

### Decision: **Hybrid**

**Why:**
- Online users get **immediate push delivery** over WebSocket (lowest latency).
- Offline users receive a **push notification** (FCM/APNs) that wakes the app, which then **pulls** the full message via REST or WebSocket on reconnect.
- Pure fan-out-on-write would be wasteful for group chats with 1,024 members: sending 1,024 WebSocket writes per message at scale is prohibitively expensive.
- For **large groups (> 100 members)**, we switch to a **fan-out-on-read** model where the message is stored once and clients pull on reconnect, reducing write amplification.

---

## 5. Storage: Relational vs. Object Storage for Media

### Decision: **S3 for media files; only metadata in PostgreSQL**

**Why:**
- Storing binary blobs in PostgreSQL causes severe **I/O contention** and makes backups enormous.
- S3 provides **virtually unlimited storage**, 11 nines of durability, and built-in CDN integration (CloudFront).
- **Pre-signed URLs** allow clients to upload directly to S3 — the app servers are not in the data path, eliminating a bandwidth bottleneck.

**Trade-offs Accepted:**
- Media delivery requires a **two-step process**: client first calls the API to get a pre-signed URL, then uploads to S3. This adds a round-trip but is the standard industry pattern.
- **Cross-region media latency** is mitigated by CloudFront CDN; the first request to a new edge may be slow, subsequent ones are cached.

---

## 6. Cassandra Partition Strategy: Single Key vs. Bucketed Key

### Decision: **Bucketed partition key (conversation_id + year-month)**

**Why:**
- A single `conversation_id` partition key would cause **unbounded partition growth** for active conversations that span years.
- Cassandra performance degrades when partitions exceed **~100 MB** (slow reads, compaction pressure).
- Adding a **time bucket** (e.g., `2026-04`) caps partition size while keeping sequential message reads efficient within a bucket.

**Trade-offs Accepted:**
- Queries that span multiple months (e.g., "load messages from the last 6 months") require **multiple queries** (one per bucket), merged client-side. This is handled transparently by the Message Service.

---

## 7. End-to-End Encryption (E2EE)

### Decision: **Signal Protocol (Double Ratchet + X3DH) — Optional, recommended**

**Why:**
- The **Signal Protocol** is the gold standard for E2EE messaging, used by WhatsApp, Signal, and iMessage.
- Provides **forward secrecy** (compromise of today's keys doesn't expose past messages) and **break-in recovery** (future messages are safe even after a key compromise).
- The server stores **only ciphertext** — even Anthropic/the platform cannot read messages.

**Trade-offs Accepted:**
- **Key management complexity**: each device generates its own keypair; if a user loses their device without a backup, messages are **irrecoverable**.
- **Multi-device delivery** requires encrypting the same message once per recipient device, increasing payload size for users with many devices.
- **Server-side search** becomes impossible — clients must index messages locally.
- **Message backup** requires either sacrificing E2EE (cloud backup) or implementing a client-controlled encrypted backup scheme.

---

## 8. Synchronous vs. Asynchronous Message Persistence

### Decision: **Asynchronous persistence via Kafka**

**Why:**
- The message send API responds **immediately** after enqueuing to Kafka, without waiting for Cassandra to acknowledge the write.
- This keeps **p99 send latency < 50 ms** even under Cassandra GC pauses or slow nodes.
- Workers consume from Kafka and write to Cassandra; if Cassandra is temporarily unavailable, messages queue safely in Kafka (7-day retention).

**Trade-offs Accepted:**
- There is a **brief window (< 1 second typically)** where a message is in Kafka but not yet in Cassandra. If the sender crashes during this window, the message is still delivered (Kafka guarantees at-least-once delivery).
- Requires careful **idempotency**: consumers use the `message_id` to deduplicate retries.

---

## Summary Table

| Decision | Choice Made | Key Trade-off |
|---|---|---|
| Real-time protocol | WebSockets | Stateful servers; need Redis pub/sub backbone |
| Message storage | Cassandra | No ad-hoc queries; schema must be planned upfront |
| Metadata storage | PostgreSQL | Vertical scaling limit; mitigated with read replicas |
| Consistency model | Eventual (messages), Strong (users) | Rare brief message ordering glitch on reconnect |
| Fan-out strategy | Hybrid push/pull | Complexity; pure push too expensive for large groups |
| Media storage | S3 + CDN | Two-step upload; first CDN request may be slow |
| Partition key | Bucketed (conv_id + month) | Multi-bucket queries for long history |
| Encryption | Signal Protocol (E2EE) | No server-side search; key management complexity |
| Write path | Async via Kafka | Sub-second window of non-durability; mitigated by at-least-once |
