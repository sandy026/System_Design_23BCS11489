# Database Design — Real-Time Chat System

---

## Overview

The system uses **two complementary databases**:

| Database | Technology | Purpose |
|---|---|---|
| **Relational (OLTP)** | PostgreSQL (CockroachDB for geo-distribution) | Users, groups, memberships, device tokens |
| **Wide-Column (NoSQL)** | Apache Cassandra | Messages, conversation timelines (high write throughput, time-series access patterns) |
| **Object Storage** | AWS S3 | Media files (images, videos, audio, documents) |
| **Cache** | Redis Cluster | Sessions, online presence, recent message cache, rate limiting |

---

## 1. Relational Schema (PostgreSQL)

### Table: `users`

```sql
CREATE TABLE users (
    user_id        UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    phone          VARCHAR(20)   UNIQUE NOT NULL,
    email          VARCHAR(255)  UNIQUE,
    name           VARCHAR(100)  NOT NULL,
    avatar_url     TEXT,
    status_message VARCHAR(255)  DEFAULT 'Hey there! I am using ChatApp.',
    password_hash  TEXT          NOT NULL,
    is_active      BOOLEAN       DEFAULT TRUE,
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_email ON users(email);
```

---

### Table: `user_presence`

```sql
CREATE TABLE user_presence (
    user_id     UUID        PRIMARY KEY REFERENCES users(user_id),
    is_online   BOOLEAN     DEFAULT FALSE,
    last_seen   TIMESTAMPTZ,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

> **Note:** Presence data is primarily maintained in **Redis** with a TTL; this table is a durable fallback.

---

### Table: `user_devices`

Stores push notification tokens for each registered device.

```sql
CREATE TABLE user_devices (
    device_id    UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID         NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    device_token TEXT         NOT NULL,
    platform     VARCHAR(10)  NOT NULL CHECK (platform IN ('ios', 'android', 'web')),
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    last_active  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_devices_user_id ON user_devices(user_id);
```

---

### Table: `user_blocks`

```sql
CREATE TABLE user_blocks (
    blocker_id  UUID        NOT NULL REFERENCES users(user_id),
    blocked_id  UUID        NOT NULL REFERENCES users(user_id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (blocker_id, blocked_id)
);
```

---

### Table: `conversations`

```sql
CREATE TABLE conversations (
    conversation_id UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    type            VARCHAR(10)  NOT NULL CHECK (type IN ('direct', 'group')),
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    last_message_at TIMESTAMPTZ,
    -- For direct chats, store both user_ids for fast lookup
    user_a_id       UUID         REFERENCES users(user_id),
    user_b_id       UUID         REFERENCES users(user_id)
);

-- Unique constraint: only one direct conversation per pair
CREATE UNIQUE INDEX idx_direct_conv
    ON conversations(LEAST(user_a_id, user_b_id), GREATEST(user_a_id, user_b_id))
    WHERE type = 'direct';
```

---

### Table: `conversation_members`

Used for group chats; also used to store per-user unread counts.

```sql
CREATE TABLE conversation_members (
    conversation_id  UUID         NOT NULL REFERENCES conversations(conversation_id),
    user_id          UUID         NOT NULL REFERENCES users(user_id),
    role             VARCHAR(10)  DEFAULT 'member' CHECK (role IN ('admin', 'member')),
    joined_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    unread_count     INT          DEFAULT 0,
    last_read_msg_id UUID,
    is_muted         BOOLEAN      DEFAULT FALSE,
    PRIMARY KEY (conversation_id, user_id)
);

CREATE INDEX idx_conv_members_user ON conversation_members(user_id);
```

---

### Table: `groups`

```sql
CREATE TABLE groups (
    group_id         UUID         PRIMARY KEY REFERENCES conversations(conversation_id),
    name             VARCHAR(100) NOT NULL,
    description      TEXT,
    avatar_url       TEXT,
    created_by       UUID         NOT NULL REFERENCES users(user_id),
    max_members      INT          DEFAULT 1024,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

---

### Table: `media_attachments`

```sql
CREATE TABLE media_attachments (
    media_id        UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    uploader_id     UUID          NOT NULL REFERENCES users(user_id),
    s3_key          TEXT          NOT NULL UNIQUE,
    cdn_url         TEXT          NOT NULL,
    thumbnail_url   TEXT,
    mime_type       VARCHAR(100)  NOT NULL,
    file_size_bytes BIGINT        NOT NULL,
    width           INT,
    height          INT,
    duration_secs   INT,
    uploaded_at     TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
```

---

## 2. NoSQL Schema (Apache Cassandra)

Cassandra is chosen for messages due to its **linear write scalability**, **tunable consistency**, and **time-series access patterns**.

### Keyspace Configuration

```cql
CREATE KEYSPACE chat
    WITH replication = {
        'class': 'NetworkTopologyStrategy',
        'us-east-1': 3,
        'eu-west-1': 3,
        'ap-south-1': 3
    }
    AND durable_writes = true;
```

---

### Table: `messages`

Partition key is `conversation_id` so all messages in a conversation are co-located. The `bucket` field (month-based) prevents unbounded partition growth.

```cql
CREATE TABLE chat.messages (
    conversation_id  UUID,
    bucket           TEXT,          -- e.g. '2026-04' (year-month)
    sent_at          TIMEUUID,      -- time-sortable UUID (stores microsecond timestamp)
    message_id       UUID,
    sender_id        UUID,
    type             TEXT,          -- 'text' | 'image' | 'video' | 'audio' | 'document'
    content          TEXT,          -- plaintext content or E2EE ciphertext
    media_id         UUID,          -- FK to media_attachments (PostgreSQL)
    reply_to_id      UUID,
    is_deleted       BOOLEAN,
    is_edited        BOOLEAN,
    edit_history     LIST<FROZEN<MAP<TEXT, TEXT>>>,
    reactions        MAP<TEXT, COUNTER>,  -- emoji → count
    PRIMARY KEY ((conversation_id, bucket), sent_at, message_id)
) WITH CLUSTERING ORDER BY (sent_at DESC)
  AND compaction = { 'class': 'TimeWindowCompactionStrategy',
                     'compaction_window_unit': 'DAYS',
                     'compaction_window_size': 1 }
  AND gc_grace_seconds = 86400;
```

---

### Table: `message_receipts`

Tracks delivery and read status per user per message.

```cql
CREATE TABLE chat.message_receipts (
    conversation_id  UUID,
    message_id       UUID,
    user_id          UUID,
    delivered_at     TIMESTAMP,
    read_at          TIMESTAMP,
    PRIMARY KEY ((conversation_id, message_id), user_id)
);
```

---

### Table: `user_conversations` (Inbox)

Denormalized view: the sorted list of a user's conversations (for rendering the inbox).

```cql
CREATE TABLE chat.user_conversations (
    user_id              UUID,
    last_message_at      TIMEUUID,
    conversation_id      UUID,
    conversation_type    TEXT,
    counterpart_id       UUID,        -- other user (direct) or group_id
    last_message_preview TEXT,
    unread_count         COUNTER,
    PRIMARY KEY (user_id, last_message_at, conversation_id)
) WITH CLUSTERING ORDER BY (last_message_at DESC);
```

---

## 3. Redis Data Structures

| Key Pattern | Type | TTL | Purpose |
|---|---|---|---|
| `presence:{user_id}` | Hash `{is_online, last_seen}` | 60 s (refreshed by heartbeat) | Online presence |
| `session:{token_id}` | String (user_id) | 3600 s | JWT session validation |
| `typing:{conv_id}:{user_id}` | String | 5 s | Typing indicator (auto-expires) |
| `conv_cache:{conv_id}` | Sorted Set (msg_id → timestamp) | 10 min | Recent messages for fast load |
| `rate_limit:{user_id}:{endpoint}` | Counter | 60 s | API rate limiting |
| `unread:{user_id}:{conv_id}` | Counter | — | Unread message badge count |

---

## 4. Entity-Relationship Summary

```
users ──< user_devices
users ──< user_blocks
users ──< conversation_members >── conversations
conversations ──< groups
conversations ──( messages [Cassandra] )
messages ──< message_receipts [Cassandra]
users ──< media_attachments
```

---

## 5. Data Access Patterns

| Operation | Database | Query / Table |
|---|---|---|
| Load user profile | PostgreSQL | `SELECT * FROM users WHERE user_id = ?` |
| List user's inbox | Cassandra | `SELECT * FROM user_conversations WHERE user_id = ?` |
| Load conversation messages | Cassandra | `SELECT * FROM messages WHERE conversation_id = ? AND bucket = ?` |
| Check online status | Redis | `GET presence:{user_id}` |
| Send message | Cassandra (write) + Redis pub/sub | Insert into `messages`, publish to channel |
| Mark as read | Cassandra | Insert into `message_receipts`, update `unread_count` |
| Search users | PostgreSQL | Full-text index on `users.name` |
| Upload media | S3 (pre-signed URL) + PostgreSQL | Insert into `media_attachments` after upload |

---

## 6. Sharding Strategy

- **PostgreSQL**: Vertical sharding by service domain (users DB, groups DB). Use **read replicas** for read-heavy queries.
- **Cassandra**: Natively sharded via **consistent hashing** on the partition key (`conversation_id + bucket`). Vnodes distribute load automatically across the cluster.
- **Redis**: **Redis Cluster** with hash slots (16,384 slots across ≥ 6 nodes).
