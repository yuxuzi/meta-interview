# ⚡️ Meta Interview: eBay-like Auction Platform — 30-Minute System Design

## 🕒 Interview Timeline (30 Minutes)
| Phase                  | Time | Goal                                 |
| ---------------------- | ---- | ------------------------------------ |
| ✅ Requirements        | 5m   | Scope, priorities, and constraints   |
| 📊 Estimations         | 3m   | Load profile, data/storage/bandwidth |
| 🏗️ High-Level Design | 8m   | Services, APIs, data flows           |
| 🔎 Deep Dive           | 10m  | Real-time bidding, auction fairness  |
| 🚀 Scaling & Tradeoffs | 4m   | Bottlenecks, edge cases, wrap-up     |

---

## 1. ✅ Requirements Clarification (5 min)

### 🔍 Key Questions:
* **Concurrency?** Peak concurrent bidders per auction? (Target: 10K+ for popular items)
* **Real-time latency?** Bid placement and broadcast SLA? (<50ms for bid confirmation)
* **Fairness rules?** Anti-sniping? Tie-breaking on simultaneous bids?
* **Consistency vs. availability?** For final seconds of auctions? (CP over AP)
* **Global scale?** Multi-region support needed?
* **Fraud prevention?** Shill bidding detection?

### ✅ Functional Requirements (Prioritized):
1. **Real-time bid placement & updates** (P0 - Core feature)
2. **Accurate auction lifecycle** (start, active, end with millisecond precision)
3. **Listing creation** (title, images, reserve, duration, categories)
4. **Auction discovery** (search, filter, recommendations)
5. **Secure payments + escrow** (hold funds, automatic release)
6. **Notifications** (outbid alerts, auction ending, payment updates)
7. **Authentication & user profiles** (reputation, bid history)

### 🚦 Non-Functional Requirements:
* **Scale**: 10M DAU, 1M concurrent auctions, 50M bids/day
* **Latency**: <50ms bid placement, <100ms bid broadcast
* **Availability**: 99.99% uptime (52 minutes downtime/year)
* **Consistency**: Strong consistency for bids, eventual for non-critical data
* **Fairness**: Deterministic winner selection, anti-sniping mechanisms
* **Security**: Fraud detection, secure payments, audit trails

---

## 2. 📊 Capacity Estimation (3 min)

### 📈 Traffic Analysis:
```
Daily Active Users: 10M
Peak hour traffic: 20% of daily (2M concurrent users)
Auction participation: 30% of users actively bidding

BID TRAFFIC:
- Bids/day: 50M → ~580 avg QPS
- Peak QPS (auction sniping): 580 × 100 = ~58K QPS
- Hot auctions: Top 1% get 50% of bids = ~29K QPS per hot auction

READ TRAFFIC:
- Auction page views: 100M/day → ~1.2K avg QPS, ~25K peak QPS
- Search queries: 20M/day → ~230 avg QPS, ~5K peak QPS
- WebSocket connections: 2M concurrent during peak
```

### 💾 Storage Requirements:
```
AUCTION DATA:
- Active auctions: 1M × 2KB = ~2GB
- Historical auctions: 365M/year × 2KB = ~730GB/year
- Auction images: 50GB/day → ~18TB/year (CDN storage)

BID DATA:
- Daily bids: 50M × 200B = ~10GB/day
- Bid history retention: 10GB × 365 = ~3.6TB/year
- Hot auction cache: 1% of active bids = ~100MB in Redis

USER DATA:
- User profiles: 100M × 1KB = ~100GB
- Watchlists: 50M users × 20 items × 50B = ~50GB
```

### 🌐 Bandwidth Requirements:
```
INCOMING:
- Bid submissions: 58K QPS × 200B = ~11.6MB/s peak
- Image uploads: 1M/day × 500KB = ~5.8MB/s avg

OUTGOING:
- Bid broadcast fanout: 58K bids × 1K watchers × 500B = ~29GB/s peak
- CDN offloading: 90% of image/static content
- WebSocket updates: 2M connections × 10 updates/min = ~6.7MB/s
```

---

## 3. 🏗️ High-Level Architecture (8 min)

### 🌟 Key Design Principles:
* **Event-driven architecture**: Kafka for reliable async processing
* **Microservices**: Domain-driven service boundaries
* **CQRS pattern**: Separate read/write paths for optimal performance
* **Cache-aside**: Redis for hot data, DB for persistence
* **Circuit breaker**: Prevent cascade failures

### 🧱 Service Architecture:
```
                    ┌─────────────────┐
                    │   Load Balancer │
                    │   (HAProxy)     │
                    └─────────┬───────┘
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                           │
┌───────▼──────┐                        ┌─────────▼────────┐
│  API Gateway │                        │ WebSocket Gateway│
│  (Kong/Envoy)│                        │   (Dedicated)    │
│ + Rate Limit │                        │  + Load Balance  │
└───────┬──────┘                        └─────────┬────────┘
        │                                         │
   HTTP REST APIs                          Real-time Connection
        │                                         │
┌───────┴──────────────────────────┐     ┌────────▼────────┐
│         Microservices            │     │  Bidding Engine │
├──────────────────────────────────┤     │  (WebSocket)    │
│┌──────┐ ┌──────┐ ┌──────────┐   │     │                 │
││Auth  │ │List  │ │Auction   │   │◄────┤• Real-time bids │
││Svc   │ │Svc   │ │Lifecycle │   │     │• Live updates   │
│└──────┘ └──────┘ │Service   │   │     │• Event broadcast│
│                  └──────────┘   │     └─────────┬───────┘
│┌──────┐ ┌──────┐ ┌──────────┐   │              │
││Search│ │Pay   │ │Notify    │◄──┼──────────────┘
││Svc   │ │Svc   │ │Service   │   │         ┌────▼────┐
│└──────┘ └──────┘ └─────┬────┘   │         │Redis    │
└──────────────────┼─────┼────────┘         │Cluster  │
                   │     │                  └─────────┘
                   │     │
        ┌──────────▼─────▼──┐
        │   Message Queue   │
        │  (Kafka Cluster)  │
        │                  │
        │ Topics:          │
        │ • bid-events     │
        │ • outbid-alerts  │
        │ • auction-ended  │
        │ • payment-due    │
        └──────────────────┘
```

### 🔗 Service Separation & Responsibilities:

**API Gateway (Kong/Envoy) - HTTP REST Services:**
```
Handles traditional request/response patterns:
• User authentication & authorization (JWT validation)
• CRUD operations (create auction, update profile)
• Search queries (find auctions, filter results) 
• Payment processing (escrow setup, transaction completion)
• Historical data queries (past bids, auction results)

Features: Rate limiting, request routing, load balancing
Protocol: HTTP/HTTPS with JSON payloads
SLA: 200ms P95 response time for complex queries
```

**WebSocket Gateway - Real-time Bidding Engine:**
```  
Dedicated infrastructure for sub-second interactions:
• Live bid placement and validation
• Real-time price updates to watchers
• Auction countdown timers
• Instant outbid notifications
• Connection management (heartbeat, reconnection)

Features: Sticky sessions, connection pooling, event broadcasting
Protocol: WebSocket with binary/JSON hybrid messaging
SLA: <50ms bid confirmation, <100ms fanout broadcast
```

**Why Separate Gateways?**
- **Performance Isolation**: WebSocket connections are long-lived and stateful
- **Scaling Strategies**: HTTP services scale horizontally, WebSocket needs sticky sessions
- **Resource Optimization**: WebSocket gateway optimized for memory and connections
- **Failure Isolation**: Real-time bidding continues even if REST APIs have issues



### 🔄 Cross-Service Communication:

**Event-Driven Architecture via Message Queue (Kafka):**

**Bidding Engine → Notification Service:**
```
Topic: "bid-events"
Event: { 
  type: "BID_PLACED", 
  auction_id: "abc123", 
  new_bidder: "user456", 
  previous_bidder: "user123", 
  amount: 150.00,
  timestamp: 1691234567890
}

Topic: "auction-events" 
Event: {
  type: "AUCTION_ENDING_SOON",
  auction_id: "abc123",
  time_remaining: 300, // 5 minutes
  current_price: 150.00,
  watchers: ["user789", "user101", ...]
}
```

**Message Queue Topics & Consumers:**
```
📨 bid-events (High Throughput: 50M events/day)
   └── Consumers: Notification Service, Analytics Service
   └── Partition Strategy: By auction_id for ordering

📨 outbid-alerts (Medium Throughput: 20M events/day)  
   └── Consumers: Notification Service, Email Service
   └── Priority: High (real-time notifications)

📨 auction-ended (Low Throughput: 1M events/day)
   └── Consumers: Payment Service, Notification Service
   └── Requires: Exactly-once delivery guarantee

📨 payment-due (Low Throughput: 200K events/day)
   └── Consumers: Payment Service, Reminder Service  
   └── Delay: 24-hour payment window trigger
```

**Why Message Queue is Critical:**
- **Decoupling**: Bidding engine doesn't wait for notification delivery
- **Reliability**: Events persisted even if notification service is down
- **Scalability**: Multiple notification service instances can process events
- **Ordering**: Kafka partitions ensure bid events processed in sequence
- **Replay**: Can replay events for failed notifications or new features

**Notification Service Event Processing:**
```
Async Event Handlers:
1. BID_PLACED → Send push notification to previous highest bidder
2. AUCTION_ENDING → Send alerts to all watchers (5min, 1min warnings)
3. AUCTION_WON → Notify winner and trigger payment flow
4. PAYMENT_DUE → Send payment reminders and deadline warnings
5. AUCTION_ENDED → Send final results to all participants
```

**Microservices ↔ Bidding Engine:**
- **Validation**: Bidding engine calls Auth service for user verification (with 1-second cache)
- **State Sync**: Auction Lifecycle service publishes start/end events to Kafka
- **Shared Redis**: Both access live auction state for consistency

### 📡 API Design & WebSocket Events:

**REST APIs (via API Gateway):**
```http
# Auction Management  
POST /v1/auctions                    # Create new auction
GET  /v1/auctions/{id}              # Get auction details  
PUT  /v1/auctions/{id}              # Update auction (owner only)

# Search & Discovery
GET  /v1/search?q=...&category=...  # Search auctions
GET  /v1/auctions/trending          # Popular/hot auctions
GET  /v1/auctions/{id}/bids         # Historical bid data

# User & Payment Operations
POST /v1/auth/login                 # User authentication
GET  /v1/users/{id}/watchlist       # User's watched auctions
POST /v1/payments/escrow            # Lock winner funds
POST /v1/payments/release           # Complete transaction
```

**WebSocket Events (via WebSocket Gateway):**
```json
// Client → Server: Place bid
{
  "type": "place_bid",
  "auctionId": "abc123",
  "amount": 120.00,
  "timestamp": 1691234567890,
  "nonce": "uuid-v4"
}

// Server → Clients: Bid update
{
  "type": "bid_update",
  "auctionId": "abc123",
  "currentPrice": 120.00,
  "bidCount": 47,
  "leadingBidder": "u123",
  "timeRemaining": 25000,
  "timestamp": 1691234567891
}

// Server → Client: Outbid notification
{
  "type": "outbid",
  "auctionId": "abc123",
  "yourBid": 115.00,
  "currentPrice": 120.00,
  "timeRemaining": 25000
}
```

---

## 4. 🔎 Deep Dive: Real-Time Bidding System (10 min)

### A. 🚀 Core Bidding Flow & Race Condition Handling

**The Critical Path: Bid Placement in <50ms**

1. **Request Validation** (5ms)
   - User authentication via JWT token
   - Check auction status (active/closed)
   - Verify user isn't auction owner
   - Balance/credit limit validation

2. **Atomic Price Update** (10ms)
   - Use Redis Lua script for atomicity
   - Compare new bid against current highest
   - Update: current_price, highest_bidder, bid_count, timestamp
   - Return success/failure immediately

3. **Event Broadcasting** (15ms)
   - WebSocket fanout to all auction watchers
   - Push to Kafka for downstream processing
   - Update search index asynchronously

4. **Persistence** (Async, non-blocking)
   - Write to primary database
   - Update user bid history
   - Trigger notifications to outbid users

**Race Condition Solutions:**
- **Atomic Operations**: Redis Lua scripts ensure no race conditions during price updates
- **Optimistic Locking**: Database writes use version numbers for conflict detection
- **Idempotency**: Each bid has unique nonce to prevent duplicate processing
- **Ordering**: All bids get nanosecond timestamps for deterministic ordering

### B. 🎯 Auction Fairness & Anti-Gaming

**1. Tie-Breaking Strategy:**
When multiple bids arrive at exactly same price:
- Earliest timestamp wins (nanosecond precision)
- If timestamps identical: lexicographically smallest user ID
- Server-side timestamps prevent client clock manipulation

**2. Anti-Sniping Protection:**
- **Soft Close**: If bid placed in last 30 seconds, extend auction by 1 minute
- **Maximum Extensions**: Limit to 5 extensions to prevent infinite auctions  
- **Notification**: Alert all watchers about time extensions

**3. Fraud Prevention:**
- **Velocity Limits**: Max 10 bids per user per minute
- **Shill Detection**: Flag suspicious patterns (same IP, rapid back-and-forth bidding)
- **Deposit Requirements**: Hold 10% of bid amount to prevent fake bids

### C. 🏎️ Performance & Scalability Optimizations

**1. Hot Data Strategy:**
- **Redis Partitioning**: Hot auctions (top 10%) get dedicated Redis instances
- **Consistent Hashing**: Distribute auctions across Redis cluster
- **Local Caching**: WebSocket servers cache auction data for 1 second

**2. WebSocket Management:**
- **Connection Pooling**: Each server handles 50K concurrent connections
- **Sticky Sessions**: Users stick to same WebSocket server
- **Graceful Degradation**: Fall back to HTTP polling if WebSocket fails
- **Message Compression**: Use binary format for 40% bandwidth reduction

**3. Database Optimization:**
- **Read Replicas**: Route auction reads to replicas
- **Write Batching**: Batch bid writes every 100ms for throughput
- **Partitioning**: Partition bids table by auction_id for parallel processing

### D. 🛡️ Consistency & Reliability

**Strong Consistency Where It Matters:**
- Bid prices and winners: Redis + immediate DB write
- Payment processing: ACID transactions required
- Auction end times: Synchronized across all servers

**Eventual Consistency for Non-Critical:**
- Search indexes: 1-2 second lag acceptable  
- User notifications: Can be delayed/retried
- Analytics data: Several minutes lag is fine

**Failure Recovery:**
- **Redis Failover**: Sentinel-based automatic failover in <30 seconds
- **Split-Brain Prevention**: Require quorum of 3 Redis Sentinels
- **Data Recovery**: Kafka event log allows full state reconstruction
- **Circuit Breakers**: Fail fast when downstream services are down

---

## 5. 🚀 Scaling & Trade-offs (4 min)

### A. 🔥 Bottlenecks & Solutions

**Primary Bottlenecks:**
1. **Redis Memory**: Hot auctions consuming excessive memory
   - **Solution**: Partition hot vs cold auctions, different Redis configurations
   - **Capacity**: 32GB Redis can handle ~100K concurrent auctions

2. **WebSocket Fanout**: Broadcasting to millions of watchers
   - **Solution**: Hierarchical fanout through multiple WebSocket servers
   - **Calculation**: 58K QPS × 1K watchers = 58M messages/sec to broadcast

3. **Database Writes**: 50M bid writes per day
   - **Solution**: Write batching, read replicas, and asynchronous persistence
   - **Capacity**: Postgres can handle ~10K writes/sec with proper indexing

### B. 🌍 Multi-Region Considerations

**Data Distribution:**
- **User Data**: Store in user's home region (GDPR compliance)
- **Hot Auctions**: Replicate to all regions with >5% watchers  
- **Payment Data**: Keep in region for regulatory requirements

**Consistency Trade-offs:**
- **Strong**: Bid prices, auction winners (single region processing)
- **Eventual**: Search results, recommendations (cross-region replication)
- **Hybrid**: Global users can watch any auction, but bid in auction's home region

### C. ⚡ Edge Cases & Failure Modes

**1. Flash Traffic Scenarios:**
- **Celebrity Auction**: 1M concurrent bidders on single auction
- **Mitigation**: Pre-provision dedicated Redis cluster, queue system for overflow
- **Graceful Degradation**: Disable non-essential features, extend bid timeouts

**2. System Failures:**
- **Redis Down**: Fall back to database-only mode (500ms latency vs 50ms)
- **WebSocket Issues**: Auto-fallback to HTTP Server-Sent Events (SSE)
- **Payment Service Down**: Queue payment processing, notify users of delay

**3. Auction End Race Conditions:**
- **Problem**: Multiple bids in final milliseconds
- **Solution**: Use distributed locks, process bids sequentially in final 10 seconds
- **Backup**: Event sourcing allows replay and correction if conflicts occur

### D. 🎯 Success Metrics

**SLA Targets:**
- Bid placement: 99.9% success rate, <50ms P99 latency
- WebSocket connections: 99.95% uptime
- Auction accuracy: 100% correct winner determination
- Payment processing: 99.99% success rate within 24 hours

**Business KPIs:**
- Support $1B+ annual GMV
- Handle 100M+ auctions per year  
- Maintain <0.1% disputed transactions
- Achieve 95%+ user satisfaction score

---

## 🎤 **Interview Wrap-Up Summary**

*"This auction platform design prioritizes three key aspects:*

1. **Real-time performance** through Redis atomic operations and WebSocket broadcasting
2. **Fairness and trust** via anti-sniping mechanisms, fraud detection, and deterministic tie-breaking
3. **Massive scale** using microservices, event-driven architecture, and intelligent caching

*The system can handle 100x traffic spikes during popular auction endings while maintaining sub-50ms bid latency and ensuring perfect auction integrity. The event-sourcing approach provides complete auditability and enables features like ML-based recommendations.*

*Key design decisions: Strong consistency for bids, eventual consistency for search, and graceful degradation during failures. Would you like me to elaborate on any specific component?"*


-------

# 🎭 Meta Interview Simulation: eBay-like Auction Platform

## 👥 **Characters:**
- **Interviewer (I)**: Senior Staff Engineer at Meta
- **Candidate (C)**: You, the interviewee

---

## ⏰ **Phase 1: Requirements Clarification (5 minutes)**

**I:** "Today we'll design an auction platform like eBay. You have 30 minutes total. Let's start with requirements - what questions do you have?"

**C:** "Great! Let me clarify the scope first. Are we focusing on the core bidding functionality, or should I include seller onboarding, fraud detection, and payments?"

**I:** "Focus on the core auction and bidding system. Assume payments exist but touch on the integration."

**C:** "Perfect. Key questions:
- What's our scale? Peak concurrent users and auctions?
- Real-time requirements? What's acceptable latency for bid placement?
- Consistency vs availability trade-offs, especially during auction endings?
- Any specific fairness rules like anti-sniping mechanisms?"

**I:** "Good questions. Let's say 10 million DAU, 1 million concurrent auctions during peak, 50 million bids per day. Bid latency should be under 50ms. Strong consistency for final bid results is critical."

**C:** "Excellent. So my functional requirements in priority order:
1. Real-time bid placement and price updates - this is P0
2. Accurate auction lifecycle management with precise timing
3. Auction creation and discovery
4. User notifications for outbids and auction endings
5. Payment integration for winners

Non-functional requirements:
- Sub-50ms bid placement latency
- 99.99% uptime 
- Strong consistency for bid ordering and final results
- Deterministic winner selection for fairness"

**I:** "Sounds right. Move to capacity planning."

---

## ⏰ **Phase 2: Capacity Estimation (3 minutes)**

**C:** "Let me break down the numbers:

**Traffic Analysis:**
- 50M bids/day = ~580 QPS average
- Peak during auction endings: 100x spike = 58K QPS
- Hot auctions get disproportionate traffic - top 1% might see 50% of bids
- Read traffic for auction watching: ~25K QPS during peak

**Storage:**
- Active auction data: 1M auctions × 2KB = ~2GB
- Bid history: 50M bids × 200B = 10GB daily, ~3.6TB annually
- Hot auction cache in Redis: ~100MB for active bidding

**Bandwidth:**
- Incoming bids: 58K QPS × 200B = ~12MB/s
- Critical challenge: Fanout broadcasting. If popular auctions have 1K+ watchers, that's 58K × 1K × 500B = ~29GB/s outbound during peaks"

**I:** "That fanout number is concerning. How would you handle it?"

**C:** "Great catch - that's exactly why we need a specialized WebSocket architecture with hierarchical fanout and regional caching. I'll show this in the design."

---

## ⏰ **Phase 3: High-Level Architecture (8 minutes)**

**C:** "I'm designing this with two distinct gateway patterns:

```
Load Balancer
├── API Gateway (REST) → Microservices
└── WebSocket Gateway → Real-time Bidding Engine
```

**Why separate gateways?**
- HTTP services are stateless, can scale horizontally
- WebSocket connections are stateful, need sticky sessions
- Different performance characteristics and resource needs
- Failure isolation - bidding continues even if search/payments have issues

**Microservices behind API Gateway:**
- Auth Service: JWT validation, user management
- Listing Service: CRUD operations for auctions
- Auction Lifecycle Service: Start/end auction management  
- Search Service: Elasticsearch for discovery
- Payment Service: Escrow and transaction processing
- Notification Service: Push notifications, emails

**Real-time Bidding Engine:**
- Handles WebSocket connections
- Validates and processes bids atomically
- Broadcasts updates to watchers
- Uses Redis for hot auction state"

**I:** "How do these services communicate? What about the notification flow when someone gets outbid?"

**C:** "Excellent question - this is where the message queue becomes critical:

**Event-Driven Architecture:**
```
Bidding Engine → Kafka → Notification Service
```

**Key Kafka Topics:**
- `bid-events`: Every bid placement (50M/day volume)
- `outbid-alerts`: Triggers for push notifications  
- `auction-ended`: Final results for payment processing

**Flow Example:**
1. User places bid via WebSocket
2. Bidding Engine updates Redis atomically
3. Broadcasts to WebSocket watchers immediately
4. Publishes event to Kafka asynchronously
5. Notification Service consumes event and sends push notification

This decouples the critical bidding path from notification delivery."

**I:** "What about the data layer? How do you ensure consistency?"

**C:** "Two-tier approach:

**Hot Path (Redis):**
- Current auction state: price, bidder, time remaining
- Atomic operations via Lua scripts prevent race conditions
- Sub-50ms read/write performance

**Persistent Layer (PostgreSQL):**
- Complete bid history and user data
- Async writes from Redis for durability
- Read replicas for historical queries

**Consistency Model:**
- Strong consistency for bid ordering within auctions
- Eventual consistency for search indexes and analytics
- Event sourcing via Kafka provides audit trail and replay capability"

---

## ⏰ **Phase 4: Deep Dive - Real-time Bidding (10 minutes)**

**I:** "Let's dive deep into the bidding system. Walk me through what happens when two users bid on the same item within milliseconds of each other."

**C:** "This is the heart of the system. Here's the atomic bidding flow:

**Step 1: Pre-validation (5ms)**
- JWT token validation (cached for 30 seconds)
- Check if auction is still active
- Verify user isn't the auction owner

**Step 2: Atomic Redis Operation (10ms)**
- Use Lua script for atomicity - critical for race conditions
- Script logic:
  - Read current price and auction status
  - Validate new bid amount > current price
  - Atomically update: price, bidder_id, timestamp, bid_count
  - Return success/failure with previous state

**Step 3: Real-time Broadcast (15ms)**
- WebSocket fanout to all auction watchers
- Message includes: new price, bidder, time remaining
- Use hierarchical broadcasting for scalability

**Step 4: Async Processing (non-blocking)**
- Publish to Kafka for notifications
- Update database persistence
- Trigger ML fraud detection

**Race Condition Handling:**
The Lua script is key here. Even if 1000 users bid simultaneously, Redis processes them sequentially. Each bid gets a nanosecond timestamp for deterministic ordering."


**I:** "How do you handle the WebSocket fanout at scale? You mentioned 29GB/s earlier."

**C:** "Multi-layered approach:

**1. Regional Distribution:**
- Deploy WebSocket gateways in multiple regions
- Route users to nearest gateway
- Cross-region replication only for popular auctions

**2. Hierarchical Fanout:**
```
Bidding Engine → Regional Message Brokers → WebSocket Servers → Clients
```

**3. Smart Connection Management:**
- Each WebSocket server handles 50K connections
- Connection pooling and multiplexing
- Heartbeat mechanism to detect dead connections
- Graceful fallback to HTTP Server-Sent Events

**4. Message Optimization:**
- Binary protocol instead of JSON (40% bandwidth reduction)
- Delta updates instead of full state
- Compress repetitive auction data

This reduces 29GB/s to manageable levels - maybe 2-3GB/s per region."



---

## ⏰ **Phase 5: Scaling & Trade-offs (4 minutes)**

**I:** "What are the main bottlenecks in this system?"

**C:** "Three critical bottlenecks:

**1. Redis Memory Pressure:**
- Problem: Hot auctions consuming excessive memory
- Solution: Partition hot vs regular auctions, different Redis configs
- Scaling: 32GB Redis handles ~100K concurrent auctions

**2. WebSocket Fanout:**
- Problem: Popular auctions with 10K+ watchers
- Solution: Regional caching, hierarchical broadcast, connection multiplexing
- Scaling: Auto-scaling WebSocket servers based on connection count

**3. Database Write Throughput:**
- Problem: 50M bid writes per day
- Solution: Write batching, read replicas, async persistence
- Scaling: PostgreSQL with proper indexing handles 10K writes/sec"

**I:** "How would you handle a celebrity auction with 1 million concurrent bidders?"

**C:** "This is an extreme scale scenario requiring special handling:

**Pre-provisioning:**
- Dedicated Redis cluster for the hot auction
- 10x normal WebSocket gateway capacity
- Pre-warm CDN and regional caches

**Load Shedding:**
- Priority queues: VIP users get guaranteed access
- Anonymous users face graceful degradation
- Display-only mode for excess traffic

**Alternative Architecture:**
- Consider lottery/sealed bid system for extreme cases
- Queue-based bidding with batch processing
- Extend auction duration to spread the load

**Graceful Degradation:**
- Disable non-essential features (fancy animations, detailed history)
- Increase bid timeout from 50ms to 500ms
- Fall back to polling instead of WebSocket for overflow users"

**I:** "What about failure scenarios?"

**C:** "Key failure modes and responses:

**Redis Cluster Failure:**
- Sentinel-based auto-failover in 30 seconds
- Fall back to database-only mode (degraded performance)
- Queue bids locally and replay when Redis recovers

**WebSocket Gateway Failure:**
- Users auto-reconnect to healthy gateways
- Sticky session data replicated across instances
- HTTP polling fallback for critical final minutes

**Message Queue Failure:**
- Local event buffering on bidding engines
- Replay events when Kafka recovers
- Critical path (bidding) continues, notifications pause

**Split-Brain Prevention:**
- Require quorum of Redis Sentinels for failover
- Distributed locks for auction end processing
- Event sourcing allows state reconstruction"

---

## 🎯 **Closing Questions**

**I:** "What metrics would you monitor for this system?"

**C:** "Four categories:

**Business Metrics:**
- Bid success rate (target: >99%)
- Auction completion rate
- Revenue per auction
- User engagement time

**Performance Metrics:**
- Bid placement latency P99 (<50ms SLA)
- WebSocket connection success rate (>99.9%)
- Message queue processing lag (<1 second)

**Reliability Metrics:**
- System uptime (99.99% target)
- Data consistency checks
- Failed bid recovery rate

**Alerting Thresholds:**
- Bid latency P95 > 100ms → Page on-call
- WebSocket drops > 10% → Immediate alert
- Payment processing errors > 1% → Critical alert"

**I:** "Last question: How would you evolve this system to add ML-based price predictions?"

**C:** "Great extension! I'd leverage the existing event-driven architecture:

**Data Pipeline:**
- Kafka already captures all bid events for ML training
- Add feature extraction service consuming bid streams
- Real-time feature store (Redis) for serving predictions

**ML Service Integration:**
- New microservice behind API Gateway
- Serve price predictions via REST API
- Cache predictions in Redis with TTL

**Gradual Rollout:**
- A/B test with small user percentage
- Monitor impact on bidding behavior
- Use circuit breakers to disable if predictions affect core bidding performance

The beauty is this doesn't touch the critical bidding path - it's purely additive using our existing event infrastructure."

**I:** "Excellent work! Any questions for me?"

**C:** "Yes - what's the biggest scaling challenge you've seen with real-time systems at Meta, and how did the team approach it?"

---

## 🎭 **Interview Debrief**

### ✅ **What Went Well:**
- **Structured approach**: Clear progression through requirements, capacity, design
- **Scale-first thinking**: Immediately identified bottlenecks and solutions
- **Trade-off discussions**: Consistency vs availability, performance vs reliability
- **Practical experience**: Showed understanding of Redis, Kafka, WebSocket challenges
- **Business awareness**: Connected technical decisions to user experience

### 🎯 **Meta-Specific Wins:**
- **Real-time systems expertise**: Critical for Meta's infrastructure
- **Event-driven architecture**: Aligns with Meta's patterns
- **Failure mode analysis**: Shows production system experience
- **Scalability mindset**: 100x spike handling, regional distribution

### 💡 **Key Takeaways:**
- **Separate concerns**: Different gateways for different use cases
- **Async where possible**: Don't block critical paths for secondary features  
- **Plan for 100x scale**: Popular events create extreme load patterns
- **Measure everything**: Metrics and alerting are first-class concerns