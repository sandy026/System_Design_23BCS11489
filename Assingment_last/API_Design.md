# API Design — Real-Time Chat System

---

## Overview

The system exposes a **REST API** for resource management (users, groups, media) and a **WebSocket API** for real-time message delivery. All REST endpoints are versioned under `/api/v1/`. Authentication uses **JWT Bearer tokens** (access token + refresh token).

---

## Authentication

### POST `/api/v1/auth/register`
Register a new user.

**Request Body**
```json
{
  "phone": "+919876543210",
  "password": "hashed_password",
  "name": "Arjun Singh",
  "otp": "482910"
}
```

**Response `201 Created`**
```json
{
  "user_id": "usr_a1b2c3",
  "access_token": "<JWT>",
  "refresh_token": "<JWT>",
  "expires_in": 3600
}
```

---

### POST `/api/v1/auth/login`
Authenticate an existing user.

**Request Body**
```json
{
  "phone": "+919876543210",
  "password": "hashed_password"
}
```

**Response `200 OK`**
```json
{
  "access_token": "<JWT>",
  "refresh_token": "<JWT>",
  "expires_in": 3600
}
```

---

### POST `/api/v1/auth/refresh`
Refresh an expired access token.

**Request Body**
```json
{ "refresh_token": "<JWT>" }
```

**Response `200 OK`**
```json
{
  "access_token": "<JWT>",
  "expires_in": 3600
}
```

---

## Users

### GET `/api/v1/users/{user_id}`
Get a user's public profile.

**Response `200 OK`**
```json
{
  "user_id": "usr_a1b2c3",
  "name": "Arjun Singh",
  "phone": "+919876543210",
  "avatar_url": "https://cdn.example.com/avatars/usr_a1b2c3.jpg",
  "status": "Hey there! I'm using ChatApp.",
  "last_seen": "2026-04-23T10:15:00Z",
  "is_online": false
}
```

---

### PUT `/api/v1/users/{user_id}`
Update the authenticated user's profile.

**Request Body**
```json
{
  "name": "Arjun S.",
  "status": "Busy",
  "avatar_url": "https://cdn.example.com/avatars/new.jpg"
}
```

**Response `200 OK`** — Returns updated user object.

---

### GET `/api/v1/users/search?q={query}`
Search users by name or phone number.

**Response `200 OK`**
```json
{
  "results": [
    { "user_id": "usr_x1y2", "name": "Arjun Kumar", "avatar_url": "..." }
  ]
}
```

---

## Conversations (Direct Messages)

### GET `/api/v1/conversations`
List all conversations for the authenticated user, sorted by last message time.

**Response `200 OK`**
```json
{
  "conversations": [
    {
      "conversation_id": "conv_001",
      "type": "direct",
      "participant": { "user_id": "usr_x1y2", "name": "Priya", "avatar_url": "..." },
      "last_message": {
        "message_id": "msg_abc",
        "content": "See you tomorrow!",
        "sent_at": "2026-04-23T09:00:00Z",
        "status": "read"
      },
      "unread_count": 0
    }
  ],
  "next_cursor": null
}
```

---

### POST `/api/v1/conversations`
Create or retrieve a direct conversation with another user.

**Request Body**
```json
{ "participant_user_id": "usr_x1y2" }
```

**Response `200 OK`** — Returns conversation object.

---

## Messages

### GET `/api/v1/conversations/{conversation_id}/messages`
Fetch paginated message history (cursor-based pagination).

**Query Params:** `before=<message_id>`, `limit=50`

**Response `200 OK`**
```json
{
  "messages": [
    {
      "message_id": "msg_abc",
      "conversation_id": "conv_001",
      "sender_id": "usr_a1b2c3",
      "type": "text",
      "content": "Hello!",
      "sent_at": "2026-04-23T08:55:00Z",
      "delivered_at": "2026-04-23T08:55:01Z",
      "read_at": "2026-04-23T08:56:00Z",
      "reactions": [],
      "reply_to": null,
      "is_deleted": false
    }
  ],
  "next_cursor": "msg_xyz"
}
```

---

### POST `/api/v1/conversations/{conversation_id}/messages`
Send a new message (fallback REST endpoint; primary path is WebSocket).

**Request Body**
```json
{
  "type": "text",
  "content": "Hello!",
  "reply_to": null,
  "client_message_id": "client_uuid_123"
}
```

**Response `201 Created`**
```json
{
  "message_id": "msg_xyz",
  "sent_at": "2026-04-23T09:00:00Z",
  "status": "sent"
}
```

---

### DELETE `/api/v1/messages/{message_id}`
Delete a message (for self or everyone).

**Query Params:** `scope=self|everyone`

**Response `200 OK`**
```json
{ "deleted": true, "scope": "everyone" }
```

---

### PUT `/api/v1/messages/{message_id}`
Edit a message (within 15 minutes of sending).

**Request Body**
```json
{ "content": "Corrected message text." }
```

**Response `200 OK`** — Returns updated message object.

---

### POST `/api/v1/messages/{message_id}/reactions`
Add or remove a reaction.

**Request Body**
```json
{ "emoji": "👍", "action": "add" }
```

**Response `200 OK`**
```json
{ "message_id": "msg_xyz", "reactions": [{ "emoji": "👍", "count": 3, "by_me": true }] }
```

---

## Group Chats

### POST `/api/v1/groups`
Create a group chat.

**Request Body**
```json
{
  "name": "Project Alpha",
  "member_ids": ["usr_x1y2", "usr_p9q8"],
  "avatar_url": null
}
```

**Response `201 Created`**
```json
{
  "group_id": "grp_001",
  "name": "Project Alpha",
  "created_at": "2026-04-23T09:00:00Z",
  "member_count": 3
}
```

---

### GET `/api/v1/groups/{group_id}`
Get group metadata and member list.

---

### PUT `/api/v1/groups/{group_id}/members`
Add or remove group members (admin only).

**Request Body**
```json
{ "action": "add|remove", "user_ids": ["usr_r5s6"] }
```

---

## Media Upload

### POST `/api/v1/media/upload-url`
Request a pre-signed S3 URL for direct media upload.

**Request Body**
```json
{
  "file_name": "photo.jpg",
  "mime_type": "image/jpeg",
  "file_size": 204800
}
```

**Response `200 OK`**
```json
{
  "upload_url": "https://s3.amazonaws.com/bucket/...",
  "media_id": "med_001",
  "expires_in": 300
}
```

After uploading the file directly to S3, include `media_id` in the message payload.

---

## WebSocket API (Real-Time)

**Endpoint:** `wss://chat.example.com/ws`  
**Auth:** Pass `Authorization: Bearer <token>` as a query param or in the connection header.

### Client → Server Events

| Event | Payload | Description |
|---|---|---|
| `message.send` | `{ conversation_id, type, content, client_message_id, reply_to? }` | Send a message |
| `message.read` | `{ conversation_id, up_to_message_id }` | Mark messages as read |
| `typing.start` | `{ conversation_id }` | Broadcast typing indicator |
| `typing.stop` | `{ conversation_id }` | Stop typing indicator |
| `presence.ping` | `{ }` | Keep-alive / heartbeat every 30 s |

### Server → Client Events

| Event | Payload | Description |
|---|---|---|
| `message.new` | Full message object | New incoming message |
| `message.updated` | `{ message_id, content, edited_at }` | Message edited |
| `message.deleted` | `{ message_id, scope }` | Message deleted |
| `message.status` | `{ message_id, status: "delivered|read" }` | Delivery / read receipt |
| `typing.indicator` | `{ conversation_id, user_id, is_typing }` | Typing indicator |
| `presence.update` | `{ user_id, is_online, last_seen }` | Online status change |
| `reaction.update` | `{ message_id, reactions[] }` | Reaction added/removed |

---

## Error Format

All errors follow a consistent schema:

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Access token is expired.",
    "request_id": "req_1a2b3c"
  }
}
```

| HTTP Code | Error Code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Invalid request payload |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `CONFLICT` | Duplicate resource |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server-side failure |

---

## Rate Limits

| Endpoint Category | Limit |
|---|---|
| Authentication | 10 requests / minute per IP |
| Messaging (REST) | 60 messages / minute per user |
| Media upload | 20 uploads / minute per user |
| Search | 30 requests / minute per user |
| General API | 300 requests / minute per user |
