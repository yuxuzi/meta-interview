# 🌟 WhatsApp-like Chat App — Standout System Design Solution

## 🚀 30-Minute Interview Agenda

| Phase | Duration | Focus |
|-------|----------|-------|
| 1. Clarify Requirements | 5 min | Gather expectations |
| 2. Capacity Estimation | 3 min | Quantify system load |
| 3. High-Level Design | 8 min | Architecture overview |
| 4. Component Deep Dive | 10 min | Zoom into real-time flow |
| 5. Scalability & Wrap-up | 4 min | Discuss bottlenecks and scaling strategy |

---

## 1. 🎯 Requirements Clarification

### ✅ Functional
- 1-to-1 and group messaging
- Real-time delivery
- Online/offline presence
- Message history (searchable)
- Authentication, multi-device sync
- Media sharing, delivery/read receipts

### 🔒 Non-Functional
- 🌍 10B msg/day, 1B users
- ⚡ <100ms latency
- ♻️ Exactly-once delivery
- 🔐 End-to-end encryption
- 💯 High availability (99.999%)

### 🤔 Clarifying Questions
- Prioritize: availability vs. consistency?
- Are read receipts eventually consistent?
- How many devices per user?
- Is encryption handled client-side?

---

## 2. 📈 Capacity Estimation

### 💬 Message Load
- 500M DAU × 20 msg/day = 10B/day
- Avg QPS = ~115K
- Peak QPS ≈ 345K

### 🧮 Storage
- 600B/msg × 10B/day = 6TB/day → 2.2PB/year
- Media: 2B × 2MB = 4PB/day → 1.46EB/year

### 📡 Bandwidth
- Inbound ≈ 200MB/s
- Outbound ≈ 1GB/s

---

## 3. 🏗 High-Level Design

```
Mobile Clients
      ↓
Load Balancer
 ↓            ↓
WebSocket   API Gateway
 Gateway        ↓
     ↓   REST Microservices
 Chat Service ├── User Service
     ↓        ├── Media Service
Message Queue └── Auth Service
     ↓            ↓
 Async Workers    Database Layer
 ├── Push      ├── PostgreSQL (User)
 ├── Delivery  ├── Cassandra (Messages)
 └── Presence  └── Redis (Presence, Dedup)
```

### 🔌 APIs
- WebSocket: `ws://chat.domain.com/connect`
- REST:
  - `POST /auth/login`
  - `POST /users`
  - `POST /messages`
  - `GET /messages?chat_id=123`
  - `POST /media/upload`

### 🗄 Database Schema
- `users(user_id, name, status, created_at)`
- `chats(chat_id, type, created_at)`
- `chat_members(chat_id, user_id)`
- `messages(msg_id, chat_id, sender_id, content, created_at, encrypted)`
- `media(media_id, user_id, url, metadata, created_at)`
- `receipts(msg_id, user_id, delivered_at, read_at)`

### ✨ Standout Design Choices
- Decoupled services for fault isolation
- Kafka for async delivery pipelines
- Redis for presence and dedup
- Cassandra TTL auto-purging old messages

---

## 4. 🔬 Deep Dive: Real-Time Messaging

### A. ✉️ Message Flow
- Client → WebSocket Gateway
- → Chat Service → Cassandra → Kafka
- ACK to sender → Kafka fanout:
  - Online users: via WebSocket
  - Offline: queue delivery worker
  - Update delivery status

### B. 📦 Exactly-Once Delivery
- Message ID = UUIDv7 (time sortable)
- Redis + TTL for deduplication
- Client-side ACK with message ID
- Retry unacked messages (exponential backoff)

### C. 🔐 End-to-End Encryption
- Double Ratchet (client-side encryption)
- Server stores encrypted blobs only
- Group chats: Sender Keys protocol
- Key rotation per thread

---

## 5. ⚙️ Scaling, Bottlenecks, Final Notes

| Component        | Bottleneck            | Mitigation                           |
|------------------|------------------------|--------------------------------------|
| WebSocket Gateway | Connection limits      | Shard by userID, load-balanced pool  |
| Cassandra         | Heavy writes           | Shard by chat_id, use TTL, geo setup |
| Media Uploads     | CDN/virus scan         | Async pipeline, external CDN         |
| Presence Updates  | Fanout overhead        | Redis pub/sub                        |

### 🛠 Tech Stack
- **Frontend**: React Native, Swift/Kotlin
- **Gateway**: NGINX + Envoy
- **Real-Time**: WebSocket + STOMP
- **Queue**: Kafka (delivery, push, metrics)
- **DB**: PostgreSQL, Cassandra, Redis
- **Media**: S3 + CloudFront
- **Monitoring**: Prometheus, Grafana, Jaeger

### 🧠 Smart Ideas to Shine
- Vector clocks or Lamport timestamps
- Redis user sharding for presence
- Chunked large media delivery
- Full-text search with MeiliSearch
- Spam detection using anomaly detection

---

## 🐚 Final Words to Interviewer
> "We start with 1M users and scale iteratively."  
> "I made deliberate trade-offs: availability > consistency."  
> "Every component is horizontally scalable and decoupled."  
> "This design is resilient, performant, and future-proof."

### ❓ Follow-Up Topics
- Secure key exchange strategy?
- Syncing state across multiple devices?
- GDPR + data retention policy?
- Scaling to 10B+ msg/hour?
- Slack-style collaboration features?


