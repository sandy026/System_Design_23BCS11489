# Requirements — Real-Time Chat System (WhatsApp-like)

---

## 1. Functional Requirements

### 1.1 User Management
- Users can **register** with phone number or email and a password.
- Users can **log in / log out** and manage their profile (name, avatar, status message).
- Users can **search** for other users by username or phone number.
- Users can **block / unblock** other users.

### 1.2 Messaging
- Users can send **one-to-one (direct) messages** in real time.
- Messages support multiple content types:
  - Plain text
  - Images, videos, audio clips
  - Documents / files
  - Emojis and reactions
- Users can **reply to**, **forward**, **edit** (within 15 minutes), and **delete** messages (for self or for everyone).
- Users can see **delivery receipts**: Sent ✓, Delivered ✓✓, Read ✓✓ (blue).
- Users can see **typing indicators** ("User is typing…").

### 1.3 Group Chat
- Users can create **group chats** with up to 1,024 members.
- Group admins can add/remove members and change group metadata (name, icon).
- Groups support all message types listed above.
- Users receive **push notifications** for new messages when the app is in the background.

### 1.4 Online Presence
- Users can see whether contacts are **online / last seen**.
- Users can choose to **hide** their last-seen status in privacy settings.

### 1.5 Media & Storage
- Media files are **uploaded to object storage** (e.g., S3) and referenced by URL.
- A **thumbnail preview** is generated for images and videos.
- Users can download media to their device.

### 1.6 (Optional) End-to-End Encryption (E2EE)
- Messages are encrypted on the sender's device and decrypted only on the recipient's device using the **Signal Protocol** (Double Ratchet + X3DH key agreement).
- The server never holds plaintext message content.

---

## 2. Non-Functional Requirements

| Category | Requirement |
|---|---|
| **Scale** | Support **500 million+ registered users** and **50 million+ daily active users (DAU)** |
| **Throughput** | Handle **~100,000 messages/second** at peak load |
| **Latency** | Message delivery p99 latency **< 200 ms** on the same continent |
| **Availability** | **99.99% uptime** (≤ 52 minutes downtime/year) |
| **Durability** | Messages persisted with **zero data loss**; replication factor ≥ 3 |
| **Consistency** | **Eventual consistency** is acceptable for read receipts; message ordering must be preserved within a conversation |
| **Security** | All data in transit over **TLS 1.3**; at-rest encryption for stored data; OWASP Top-10 mitigations |
| **Scalability** | Horizontally scalable; no single point of failure |
| **Compliance** | GDPR-compliant data handling; configurable data-retention policies |
| **Media** | Support attachments up to **100 MB** per file; auto-compress images on upload |

---

## 3. Capacity Estimation

| Metric | Estimate |
|---|---|
| DAU | 50 million |
| Avg. messages sent per user/day | 40 |
| Total messages/day | 2 billion |
| Peak QPS (messages) | ~100,000 |
| Avg. message size (text) | 200 bytes |
| Storage/day (text only) | ~400 GB |
| Media messages (10% of total) | 200 million/day |
| Avg. media size | 500 KB |
| Storage/day (media) | ~100 TB |
| Total storage/year (with replication ×3) | ~110 PB |

---

## 4. Constraints & Assumptions

- The system must work on **mobile (iOS/Android)** and **Web** clients.
- A user may be logged in on **up to 4 devices** simultaneously.
- Messages are **retained for 3 years** by default (configurable).
- Phone number / email verification is required during registration (OTP flow).
- The system is deployed on a **multi-region cloud** infrastructure (e.g., AWS us-east-1, eu-west-1, ap-south-1).
