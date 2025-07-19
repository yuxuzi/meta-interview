
# eBay-like Auction Platform System Design - 30-Minute Interview Prep

## 🎯 Interview Timeline (30 minutes)

* **Requirements Clarification (5 min)** : Understand scope and constraints
* **Capacity Estimation (3 min)** : Back-of-envelope calculations
* **High-Level Design (8 min)** : Core architecture and APIs
* **Deep Dive (10 min)** : Focus on 1-2 critical components
* **Scaling & Wrap-up (4 min)** : Bottlenecks and solutions

---

## 1. Requirements Clarification (5 minutes)

### Key Questions to Ask the Interviewer:

* "What's the scale? How many users, auctions, and bids per day?"
* "How many concurrent users bidding on popular auctions?"
* "Should we prioritize consistency or availability for bid placement?"
* "What's the acceptable latency for real-time bid updates?"
* "Do we need to handle last-second bidding spikes?"

### Functional Requirements (Priority Order):

1. **Real-time bid placement and updates** (handle bids within milliseconds of closing)
2. **Auction lifecycle management** (scheduled → active → ended)
3. **Item listing with auction settings** (duration, starting price, reserve)
4. **Auction search and discovery**
5. **Payment processing and escrow**
6. **Notifications** (outbid alerts, auction won, payment status)
7. **User registration and authentication**
8. **Seller/buyer profiles and ratings**

### Non-Functional Requirements:

* **Scalability** : 10M users, 1M active auctions, 50M bids/day
* **Low Latency** : <50ms for bid placement, <100ms for real-time updates
* **High Availability** : 99.99% uptime (auctions can't go down)
* **Strong Consistency** : No duplicate bids, accurate bid ordering
* **Fairness** : Handle simultaneous bids at auction close
* **Security** : Prevent bid manipulation, secure payments

---

## 2. Capacity Estimation (3 minutes)

### 📊 Back-of-Envelope Calculations

#### Traffic Estimation:

```
Daily Active Users (DAU): 10M
Active auctions: 1M concurrent
Bids per auction: 50 average
Total bids per day: 50M

QPS Calculation:
- Average bid QPS: 50M ÷ 86,400 = ~580 QPS
- Peak QPS (last-minute bidding): 580 × 100 = 58K QPS
- Read QPS (auction viewing): 580 × 20 = 11.6K QPS

Payment transactions: 200K auctions end daily = 200K payments
```

#### Storage Estimation:

```
Auction data: 1KB per auction × 1M = 1GB
Bid data: 200 bytes per bid × 50M/day = 10GB/day
User data: 2KB per user × 100M users = 200GB
Images: 5 images/auction × 2MB = 10MB per auction = 10TB for 1M auctions

Annual storage:
- Bid history: 10GB × 365 = 3.6TB/year
- Image storage: 10TB active + CDN
```

#### Bandwidth Estimation:

```
Incoming bids: 58K QPS × 200 bytes = 11.6MB/s peak
Outgoing updates: 58K × 500 bytes × 100 watchers = 2.9GB/s peak
Image delivery: CDN handles 100GB/day
```

---

## 3. High-Level Design (8 minutes)

### Core Services Architecture:

```
[Web/Mobile Apps] ←→ [Load Balancer] ←→ [API Gateway]
                           ↓                   ↓
                   [WebSocket Gateway]  [REST Microservices]
                           ↓            ├── User Service
                   [Auction Service]    ├── Listing Service
                           ↓            ├── Search Service
                   [Bidding Service]    ├── Payment Service
                           ↓            └── Notification Service
                   [Message Queue]             ↓
                           ↓            [Database Layer]
                   [Async Workers]      ├── PostgreSQL
                   ├── Scheduler        ├── Redis
                   ├── Payment          └── Elasticsearch
                   └── Notification
```

### Key APIs:

#### REST APIs:

```
# Authentication
POST /api/v1/auth/login
GET /api/v1/users/{id}

# Auctions
POST /api/v1/auctions
GET /api/v1/auctions/{id}
GET /api/v1/auctions/search?q=laptop&category=electronics

# Bidding
POST /api/v1/auctions/{id}/bids
GET /api/v1/auctions/{id}/bids

# Payments
POST /api/v1/payments/initiate
PUT /api/v1/payments/{id}/confirm
```

#### WebSocket Events:

```
// Client → Server
{
  "type": "place_bid",
  "auctionId": "123",
  "amount": 150.00,
  "bidId": "bid_456"
}

// Server → Client
{
  "type": "bid_update",
  "auctionId": "123",
  "currentPrice": 150.00,
  "bidder": "user_789",
  "timeLeft": 300000
}
```

### Service Communication:

#### Synchronous (REST/gRPC):

* User authentication
* Auction listing fetch
* Payment initiation

#### Asynchronous (Message Queue):

* New bid placed → broadcast to watchers
* Auction ended → notify winner + trigger payment
* Payment failed → retry/alert
* Outbid notification → push notification

### Database Design:

#### Auction Service (PostgreSQL):

sql

```sql
CREATETABLE auctions (
    id UUID PRIMARYKEY,
    seller_id BIGINTNOTNULL,
    title VARCHAR(255)NOTNULL,
    description TEXT,
    starting_price DECIMAL(10,2),
    current_price DECIMAL(10,2),
    reserve_price DECIMAL(10,2),
statusVARCHAR(20)DEFAULT'scheduled',
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    created_at TIMESTAMPDEFAULTNOW()
);

CREATEINDEX idx_auctions_status_end_time ON auctions(status, end_time);
CREATEINDEX idx_auctions_seller ON auctions(seller_id);
```

#### Bidding Service (Time-series optimized):

sql

```sql
CREATETABLE bids (
    id UUID PRIMARYKEY,
    auction_id UUID NOTNULL,
    bidder_id BIGINTNOTNULL,
    amount DECIMAL(10,2)NOTNULL,
timestampTIMESTAMPDEFAULTNOW(),
statusVARCHAR(20)DEFAULT'active'
);

CREATEINDEX idx_bids_auction_timestamp ON bids(auction_id,timestampDESC);
CREATEINDEX idx_bids_bidder ON bids(bidder_id);

-- Redis for real-time auction state
Key: auction:{auction_id}
Value: {
"current_price": 150.00,
"highest_bidder": "user_789",
"bid_count": 25,
"watchers": 150
}
```

---

## 4. Deep Dive Topics (10 minutes)

Choose 1-2 topics based on interviewer's interest:

### A. Real-time Bidding System

#### Bid Processing Flow:

```
1. Client places bid via WebSocket
2. WebSocket Gateway forwards to Bidding Service
3. Bidding Service validates bid (amount, user funds, auction status)
4. If valid:
   - Update auction state in Redis
   - Store bid in database
   - Publish bid event to message queue
5. WebSocket Gateway broadcasts update to all watchers
6. Async workers handle notifications
```

#### Race Condition Handling:

```
Problem: Multiple simultaneous bids at same price
Solution: 
- Use Redis atomic operations (WATCH/MULTI/EXEC)
- Timestamp-based ordering for tie-breaking
- Optimistic locking with retry mechanism

Redis Lua Script:
if redis.call('get', 'auction:123:price') < bid_amount then
    redis.call('set', 'auction:123:price', bid_amount)
    redis.call('set', 'auction:123:winner', user_id)
    return 'success'
else
    return 'outbid'
end
```

### B. Auction Timing and Lifecycle

#### Scheduler Service Architecture:

```
1. Dedicated Scheduler Service
2. Uses Redis sorted sets for timing:
   Key: auction_schedule
   Score: timestamp
   Value: auction_id

3. Polling mechanism every second:
   - Check Redis for auctions to start/end
   - Process state transitions
   - Handle failures with retry logic
```

#### Auction State Machine:

```
SCHEDULED → (start_time) → ACTIVE → (end_time) → ENDED → COMPLETED

Transitions:
- SCHEDULED to ACTIVE: Enable bidding, notify watchers
- ACTIVE to ENDED: Stop new bids, determine winner
- ENDED to COMPLETED: Process payment, transfer ownership
```

#### Handling Last-Second Bidding:

```
Anti-Sniping Strategy:
1. Extend auction by 5 minutes if bid placed in final 30 seconds
2. Maximum extensions: 3 times
3. Notify all watchers of time extension

Implementation:
- Check remaining time before accepting bid
- Update end_time in database and Redis
- Broadcast time extension event
```

### C. Payment Processing

#### Payment Flow:

```
1. Auction ends → Winner determined
2. Payment Service creates escrow transaction
3. Charge buyer's payment method
4. Hold funds in escrow until item delivery
5. Seller ships item → tracking provided
6. Buyer confirms receipt OR auto-confirm after 7 days
7. Release funds to seller
```

#### Payment Consistency:

```
Challenges:
- Payment failures
- Double charging
- Concurrent payment attempts

Solutions:
- Idempotent payment operations using payment_id
- Distributed transactions with saga pattern
- Compensation actions for failed payments
- Payment status state machine
```

### D. High-Availability Architecture

#### Fault Tolerance:

```
1. Bidding Service:
   - Multiple replicas behind load balancer
   - Redis cluster for auction state
   - Database read replicas

2. WebSocket Gateway:
   - Horizontal scaling with session affinity
   - Graceful failover for connections
   - Message queue for guaranteed delivery

3. Scheduler Service:
   - Leader election using Redis/ZooKeeper
   - Backup scheduler for failover
   - Audit logs for missed events
```

---

## 5. Scaling & Wrap-up (4 minutes)

### Bottlenecks & Solutions:

#### 1. Bidding Service Bottleneck:

* **Problem** : High write load, race conditions
* **Solution** : Redis cluster, optimistic locking, bid queues

#### 2. WebSocket Connection Limits:

* **Problem** : Millions of concurrent watchers
* **Solution** : Connection pooling, regional gateways, selective updates

#### 3. Database Write Load:

* **Problem** : 50M bids/day write load
* **Solution** : Sharding by auction_id, write-through cache, batch inserts

#### 4. Search Performance:

* **Problem** : Complex auction searches
* **Solution** : Elasticsearch cluster, search result caching

### Technology Stack:

```
Frontend: React/Angular, iOS/Android apps
API Gateway: Kong, AWS API Gateway
Real-time: WebSockets, Server-Sent Events
Message Queue: Apache Kafka, AWS SQS
Databases: PostgreSQL (ACID), Redis (cache), Elasticsearch (search)
Payments: Stripe, PayPal integration
Monitoring: Prometheus, Grafana, DataDog
CDN: CloudFront for images
```

### Final Architecture Diagram:

```
[Web/Mobile Clients]
        ↓
[Global Load Balancer + CDN]
        ↓
[Regional Data Centers]
    ↓           ↓
[WebSocket     [API Gateway]
 Gateway]           ↓
    ↓         [Microservices]
[Auction      ├── User Service
 Service]     ├── Listing Service
    ↓         ├── Search Service
[Bidding      ├── Payment Service
 Service]     └── Notification Service
    ↓               ↓
[Message Queue] [Databases]
    ↓           ├── PostgreSQL Cluster
[Workers]       ├── Redis Cluster
├── Scheduler   └── Elasticsearch
├── Payment
└── Notification
```

---

## 🎯 Interview Tips:

1. **Start with basic auction flow** : Create → List → Bid → Win → Pay
2. **Focus on real-time aspects** : Bidding is the core differentiator
3. **Address fairness** : How to handle simultaneous bids
4. **Consider edge cases** : Network partitions, clock synchronization
5. **Discuss trade-offs** : Consistency vs availability for bidding
6. **Scale incrementally** : 1K auctions → 1M auctions
7. **Security mindset** : Prevent bid manipulation and fraud

## 🔍 Common Follow-up Questions:

* "How do you prevent auction sniping?"
* "What happens if payment fails after auction ends?"
* "How do you handle time zone differences?"
* "How would you implement proxy bidding (auto-bid up to max)?"
* "How do you detect and prevent fake bidding?"
* "What's your disaster recovery plan for active auctions?"

Remember: Auction systems are fundamentally about **timing, fairness, and money** - emphasize how your design handles these critical aspects!
