# Scaling — Real-Time Chat System

---

## 1. Overview

The system must handle **50 million DAU**, **~100,000 messages/second** at peak, and **millions of concurrent WebSocket connections**. The scaling strategy addresses every layer: ingress, application, messaging, storage, and content delivery.

---

## 2. Load Balancing

### 2.1 Global Load Balancing
- Use **AWS Global Accelerator** (or Cloudflare) to route users to the nearest regional deployment (us-east-1, eu-west-1, ap-south-1).
- **Anycast routing** minimises round-trip time for WebSocket handshakes.

### 2.2 Regional Load Balancers
- **Layer-4 (TCP) NLB** in front of WebSocket (Chat) servers — NLBs preserve long-lived TCP connections without terminating them.
- **Layer-7 (HTTP) ALB** in front of REST API servers — enables path-based routing, WAF integration, and header inspection.

### 2.3 WebSocket Sticky Sessions
- The L4 NLB uses **IP-hash-based routing** to ensure a client's reconnect lands on the same Chat server (avoids session loss).
- Chat servers store connection state in **Redis** so that any server can handle re-connections if the original server restarts.

### 2.4 Health Checks & Auto-Scaling
- All service tiers are managed by **Kubernetes HPA (Horizontal Pod Autoscaler)** based on CPU, memory, and custom metrics (active WebSocket connections, message queue lag).
- Target: scale out in < 60 seconds when queue lag exceeds 5,000 messages.

---

## 3. WebSocket Connection Management

### 3.1 Connection Scale
- Each Chat server process handles **~50,000 concurrent WebSocket connections** (Node.js / Go event loop).
- 50 million DAU × assumed 20% concurrency = **~10 million concurrent connections**.
- Requires **~200 Chat server pods** (each handling 50 K connections) running across the cluster.

### 3.2 Pub/Sub Backbone
- When Server A receives a message from User X, it publishes to a **Redis Pub/Sub channel** keyed by `conversation_id`.
- All Chat servers subscribed to that channel (because they hold connections for conversation participants) receive the event and push it over WebSocket to connected recipients.
- For very high-volume conversations (large groups), **Apache Kafka** replaces Redis pub/sub as the fan-out backbone.

### 3.3 Offline Delivery
- If a recipient is **offline**, the message is stored in Cassandra and a **push notification** is dispatched via:
  - **FCM** (Android)
  - **APNs** (iOS)
  - **Web Push** (browser)

---

## 4. Message Queue & Async Processing

### 4.1 Kafka as the Message Backbone

```
[Chat Server]  →  Kafka Topic: chat.messages  →  [Message Persistence Worker]
                                               →  [Notification Worker]
                                               →  [Delivery Receipt Worker]
                                               →  [Analytics Worker]
```

- **Topic:** `chat.messages` partitioned by `conversation_id` → preserves ordering within a conversation.
- **Replication factor:** 3, with `min.insync.replicas=2` for durability.
- **Retention:** 7 days on Kafka (workers consume immediately; Cassandra is the durable store).

### 4.2 Benefits
- **Decouples** message send from all downstream processing.
- **Backpressure-safe**: If Cassandra is slow, the Kafka queue absorbs the burst without dropping messages.
- **Exactly-once semantics** using Kafka transactions + idempotent consumer logic.

---

## 5. Caching Strategy

### 5.1 Cache Layers

| Layer | Technology | What is Cached | TTL |
|---|---|---|---|
| CDN | CloudFront | Static assets, media thumbnails | 30 days |
| API Gateway | Redis | Auth token validation, rate-limit counters | 60 s / 1 min |
| App Cache | Redis | Recent messages per conversation (last 50) | 10 min |
| App Cache | Redis | User profile data | 5 min |
| App Cache | Redis | Online presence (`presence:{user_id}`) | 60 s TTL (refreshed by heartbeat) |
| DB Read Replica | PostgreSQL Read Replica | Complex SQL queries (group membership, user search) | — |

### 5.2 Cache Invalidation
- **Write-through cache**: On message send, write to both Cassandra and the Redis conversation cache atomically.
- **Event-driven invalidation**: Profile updates publish a `user.updated` Kafka event; consumers purge the Redis key.
- **TTL-based expiry**: Presence keys auto-expire after 60 s; clients send heartbeats every 30 s to refresh.

### 5.3 Cache Eviction
- Redis uses **allkeys-lru** eviction policy.
- Redis Cluster is sized at **≥ 3 TB total** across nodes for conversation caches.

---

## 6. Database Scaling

### 6.1 PostgreSQL
| Technique | Details |
|---|---|
| **Read replicas** | 3 read replicas per region (users, groups). All reads go to replicas; writes go to primary. |
| **Connection pooling** | PgBouncer in transaction mode (pool of 500 connections per pod) |
| **Vertical scaling** | Primary: r6g.4xlarge (128 GB RAM) with NVMe SSDs |
| **Partitioning** | `user_devices` and `conversation_members` partitioned by `user_id` hash for large tables |

### 6.2 Cassandra
| Technique | Details |
|---|---|
| **Horizontal scaling** | Add nodes — Cassandra rebalances automatically via consistent hashing |
| **Partition sizing** | Time-bucketed partition key (`conversation_id + bucket`) keeps partitions < 100 MB |
| **Compaction** | `TimeWindowCompactionStrategy` (TWCS) for time-series data — minimises read amplification |
| **Consistency levels** | Writes: `LOCAL_QUORUM`; Reads: `LOCAL_ONE` (with read-repair for accuracy) |
| **Multi-DC replication** | `NetworkTopologyStrategy` with RF=3 per region |

### 6.3 Sharding
- **Functional sharding**: User service DB, Group service DB, and Analytics DB are separate clusters.
- **Horizontal sharding (PostgreSQL)**: If a single users table exceeds 500 GB, shard by `user_id % N` using **Citus** extension.

---

## 7. Media Storage Scaling

- Media files are stored in **AWS S3** (virtually unlimited, 11 nines durability).
- **Direct upload via pre-signed URLs**: clients upload directly to S3 — bypasses app servers entirely.
- **CloudFront CDN** serves media at the edge in 300+ PoPs globally.
- **Lambda@Edge** generates thumbnails on first request and caches at the CDN layer.
- **S3 Intelligent-Tiering** automatically moves infrequently accessed media to cheaper storage classes.

---

## 8. Microservices & Independent Scaling

Each service scales independently based on its own bottlenecks:

| Service | Bottleneck | Scale-Out Trigger |
|---|---|---|
| **Chat (WebSocket)** | CPU / connections | WebSocket connections > 40 K/pod |
| **REST API** | CPU / latency | CPU > 70% or p99 latency > 300 ms |
| **Message Worker** | Kafka lag | Consumer group lag > 10 K messages |
| **Notification Worker** | FCM/APNs throughput | Queue depth > 50 K |
| **Media Service** | Upload bandwidth | CPU > 60% |
| **Search Service** | Query latency | p95 > 200 ms |

---

## 9. Multi-Region Deployment

```
Region: us-east-1 (primary)      Region: eu-west-1         Region: ap-south-1
├── API / Chat servers             ├── API / Chat servers    ├── API / Chat servers
├── Kafka cluster (3 brokers)      ├── Kafka cluster         ├── Kafka cluster
├── Cassandra (RF=3)               ├── Cassandra (RF=3)      ├── Cassandra (RF=3)
├── PostgreSQL primary             ├── PG read replica       ├── PG read replica
└── Redis Cluster                  └── Redis Cluster         └── Redis Cluster
```

- **Cassandra** cross-region replication is asynchronous; consistency within a region is `LOCAL_QUORUM`.
- **PostgreSQL** uses logical replication to replicate the users table to all regions.
- In case of regional failure, traffic is rerouted via Global Accelerator to the next nearest region within **< 30 seconds**.

---

## 10. Rate Limiting & DDoS Protection

- **API Gateway level**: Token bucket algorithm in Redis (`rate_limit:{user_id}:{endpoint}`).
- **Global**: AWS Shield Advanced + Cloudflare Magic Transit for volumetric DDoS.
- **Application**: Per-IP and per-user rate limits enforced at the API Gateway before requests reach application pods.
- **WebSocket**: Clients are limited to **100 messages/minute** per connection; violations trigger connection termination.
