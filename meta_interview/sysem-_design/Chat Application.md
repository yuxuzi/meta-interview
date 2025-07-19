
# WhatsApp-like Chat Application System Design - 30-Minute Interview Prep

## 🎯 Interview Timeline (30 minutes)

* **Requirements Clarification (5 min)** : Understand scope and constraints
* **Capacity Estimation (3 min)** : Back-of-envelope calculations
* **High-Level Design (8 min)** : Core architecture and APIs
* **Deep Dive (10 min)** : Focus on 1-2 critical components
* **Scaling & Wrap-up (4 min)** : Bottlenecks and solutions

---

## 1. Requirements Clarification (5 minutes)

### Key Questions to Ask the Interviewer:

* "What's the scale? How many users and messages per day?"
* "Should we prioritize availability or consistency?"
* "What's the acceptable message delivery latency?"
* "What's the read/write ratio?"
* "Do we need end-to-end encryption?"

### Functional Requirements (Priority Order):

1. **1-to-1 and group messaging**
2. **Real-time message delivery**
3. **User registration and authentication**
4. **Online/offline status**
5. **Message history and persistence**
6. **Media sharing (images, videos)**
7. **Message delivery receipts**
8. **Multi-device synchronization**

### Non-Functional Requirements:

* **Scalability** : 1B users, 10B messages/day
* **Low Latency** : <100ms message delivery
* **High Availability** : 99.9% uptime
* **Consistency** : No duplicate or lost messages
* **Security** : End-to-end encryption
* **Reliability** : Messages delivered exactly once

---

## 2. Capacity Estimation (3 minutes)

### 📊 Back-of-Envelope Calculations

#### Traffic Estimation:

```
Daily Active Users (DAU): 500M
Messages per user per day: 20
Total messages per day: 10B

QPS Calculation:
- Average QPS: 10B ÷ 86,400 = ~115K QPS
- Peak QPS: 115K × 3 = 345K QPS
- Read QPS: 345K × 5 = 1.7M QPS (assuming 5:1 read/write ratio)
```

#### Storage Estimation:

```
Message size: 100 bytes (text) + 500 bytes (metadata) = 600 bytes
Daily storage: 10B × 600 bytes = 6TB/day
Annual storage: 6TB × 365 = 2.2PB/year
With 5-year retention: ~11PB total

Media files: 20% of messages have media
Daily media messages: 10B × 0.2 = 2B messages/day
Average media size: 2MB
Daily media storage: 2B × 2MB = 4PB/day
Annual media storage: 4PB × 365 = 1.46EB/year
```

#### Bandwidth Estimation:

```
Incoming: 345K QPS × 600 bytes = 200MB/s
Outgoing: 1.7M QPS × 600 bytes = 1GB/s
```

---

## 3. High-Level Design (8 minutes)

### Core Services Architecture:

```
[Mobile Apps] ←→ [Load Balancer] ←→ [API Gateway (REST)]
                      ↓                      ↓
              [WebSocket Gateway]    [Microservices (REST)]
                      ↓              ├── User Service
              [Chat Service]         ├── Media Service
                      ↓              └── Auth Service
              [Message Queue]              ↓
                      ↓              [Databases]
              [Async Workers]        ├── PostgreSQL
              ├── Notification       ├── Cassandra
              ├── Delivery           └── Redis
              └── Presence
```

### Key APIs:

#### REST APIs:

```
POST /api/v1/auth/login
GET /api/v1/users/me
POST /api/v1/chats
GET /api/v1/chats/{chatId}/messages
POST /api/v1/chats/{chatId}/messages
PUT /api/v1/users/me/status
```

#### WebSocket Events:

```
// Client → Server
{
  "type": "send_message",
  "chatId": "123",
  "content": "Hello!",
  "messageId": "msg_456"
}

// Server → Client
{
  "type": "new_message",
  "chatId": "123",
  "senderId": "user_789",
  "content": "Hello!",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Database Design:

#### User Service (PostgreSQL):

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    username VARCHAR(50),
    profile_picture_url TEXT,
    last_seen TIMESTAMP,
    status VARCHAR(20) DEFAULT 'offline',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_phone ON users(phone_number);
```

#### Chat Service Database (Cassandra):

```sql
CREATE TABLE messages (
    chat_id UUID,
    message_id UUID,
    sender_id BIGINT,
    content TEXT,
    message_type VARCHAR(20),
    timestamp TIMESTAMP,
    delivered_to SET<BIGINT>,
    read_by SET<BIGINT>,
    PRIMARY KEY (chat_id, timestamp, message_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);

CREATE TABLE user_chats (
    user_id BIGINT,
    chat_id UUID,
    chat_type VARCHAR(20),
    last_message_time TIMESTAMP,
    PRIMARY KEY (user_id, last_message_time, chat_id)
) WITH CLUSTERING ORDER BY (last_message_time DESC);
```

---

## 4. Deep Dive Topics (10 minutes)

Choose 1-2 topics based on interviewer's interest:

### A. Real-time Message Delivery

#### WebSocket vs REST Separation:

```
WebSocket Usage (Real-time chat only):
- Send/receive messages in real-time
- Typing indicators
- Presence updates
- Message delivery receipts

REST API Usage (Standard operations):
- User registration/login
- Fetch chat history
- Upload media files
- Update user profile
- Create/join groups
```

#### Message Flow:

```
1. User A sends message via WebSocket
2. WebSocket Gateway connects directly to Chat Service
3. Chat Service:
   - Stores message in database
   - Publishes event to Message Queue
   - Returns acknowledgment to sender
4. Queue workers process:
   - Push notifications (for offline users)
   - Delivery tracking
   - Analytics
5. Chat Service delivers to online recipients via WebSocket
```

### B. Message Consistency & Ordering

#### Challenges:

* Messages arriving out of order
* Duplicate messages
* Network partitions

#### Solutions:

```
1. Message Ordering:
   - Use timestamp + sequence number
   - Implement vector clocks for distributed ordering
   
2. Exactly-Once Delivery:
   - Idempotent message processing
   - Deduplication using message_id
   
3. Acknowledgment System:
   - Client sends ACK upon message receipt
   - Server retries unacknowledged messages
```

### C. Group Chat Fanout

#### Challenges:

* Large groups (1000+ members)
* Message broadcasting efficiency
* Consistent read receipts

#### Message Queue Usage:

```
Queue Purpose:
- Decouple real-time processing from business logic
- Handle notification fanout asynchronously
- Ensure message delivery guarantees
- Process background tasks (delivery receipts, analytics)

Queue Topics:
- message.sent → triggers delivery workers
- user.presence → updates presence service
- notification.push → sends push notifications
- media.process → handles file uploads
```

### D. Presence Service

#### Implementation:

```
1. User Status Updates:
   - WebSocket heartbeat every 30s
   - Status changes (online/offline/typing)
   - Last seen timestamp

2. Presence Broadcasting:
   - Only to user's contacts
   - Batch updates to reduce network traffic
   - Cache recent presence data
```

---

## 5. Scaling & Wrap-up (4 minutes)

### Bottlenecks & Solutions:

#### 1. WebSocket Gateway Bottleneck:

* **Problem** : Single point of failure, connection limits
* **Solution** : Horizontal scaling, load balancing, connection pooling

#### 2. Message Storage Bottleneck:

* **Problem** : Write-heavy workload, storage growth
* **Solution** : Sharding by chat_id, data archiving, read replicas

#### 3. Real-time Delivery Bottleneck:

* **Problem** : High fanout for popular groups
* **Solution** : Message queues, worker pools, caching

### Technology Stack:

```
Frontend: React Native, Swift/Kotlin
API Gateway: Kong, AWS API Gateway
Real-time: WebSockets, Socket.io
Message Queue: Apache Kafka, RabbitMQ
Databases: PostgreSQL (user data), Cassandra (messages)
Cache: Redis (sessions, presence)
CDN: CloudFront (media files)
Push Notifications: Firebase Cloud Messaging
Monitoring: Prometheus, Grafana, ELK stack
```

### Final Architecture Diagram:

```
[Mobile Clients]
        ↓
[Load Balancer]
    ↓       ↓
[WebSocket    [API Gateway]
 Gateway]          ↓
    ↓        [REST Microservices]
[Chat         ├── User Service
 Service]     ├── Media Service
    ↓        └── Auth Service
[Message            ↓
 Queue]       [Database Layer]
    ↓        ├── PostgreSQL
[Async        ├── Cassandra
 Workers]     └── Redis
├── Push
├── Delivery
└── Presence
```

---

## 🎯 Interview Tips:

1. **Start simple** : Basic 1-to-1 messaging, then add complexity
2. **Ask clarifying questions** : Show you understand requirements matter
3. **Think aloud** : Explain your thought process
4. **Consider trade-offs** : Discuss CAP theorem implications
5. **Scale incrementally** : Start with 1M users, then scale to 1B
6. **Handle edge cases** : Network failures, concurrent updates
7. **Security considerations** : Authentication, encryption, rate limiting

## 🔍 Common Follow-up Questions:

* "How would you handle message encryption?"
* "What happens when a user switches devices?"
* "How do you prevent spam and abuse?"
* "How would you implement message search?"
* "What's your disaster recovery strategy?"

Remember: There's no single "correct" solution. Focus on demonstrating systematic thinking, understanding of trade-offs, and ability to design scalable systems!
