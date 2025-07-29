
# ⚡️ Meta Interview: eBay-like Auction Platform — 30-Minute System Design

## 🕒 Interview Timeline (30 Minutes)

| Phase                  | Time | Goal                                 |
| ---------------------- | ---- | ------------------------------------ |
| ✅ Requirements         | 5m   | Scope, priorities, and constraints   |
| 📊 Estimations         | 3m   | Load profile, data/storage/bandwidth |
| 🏗️ High-Level Design  | 8m   | Services, APIs, data flows           |
| 🔎 Deep Dive           | 10m  | Real-time bidding, auction fairness  |
| 🚀 Scaling & Tradeoffs | 4m   | Bottlenecks, edge cases, wrap-up     |

---

## 1. ✅ Requirements Clarification (5 min)

### 🔍 Key Questions:

* **Concurrency?** Peak concurrent bidders per auction?
* **Real-time latency?** Bid placement and broadcast SLA?
* **Fairness rules?** Anti-sniping? Tie-breaking on simultaneous bids?
* **Consistency vs. availability?** For final seconds of auctions?

### ✅ Functional Requirements (Prioritized):

1. Real-time bid placement & updates
2. Accurate auction lifecycle (start, active, end)
3. Listing creation (title, images, reserve, duration)
4. Auction discovery (search, filter)
5. Secure payments + escrow
6. Notifications (outbid, won, payment updates)
7. Authentication & profiles

### 🚦 Non-Functional Requirements:

* 10M DAU, 1M live auctions, 50M bids/day
* <50ms bid latency; 99.99% uptime
* Strong consistency for bids
* Fairness: deterministic winner selection
* Secure, auditable, and idempotent operations

---

## 2. 📊 Capacity Estimation (3 min)

### QPS & Load:

```
- Bids/day: 50M → ~580 avg QPS
- Peak QPS (sniping): 580 × 100 = ~58K QPS
- Auction reads: 11K+ QPS
- Payments: ~200K/day
```

### Storage:

```
- Auctions: 1M × 1KB = ~1GB
- Bids: 10GB/day × 365 = ~3.6TB/year
- Images: 10TB (served via CDN)
```

### Bandwidth:

```
- Incoming bids: 11.6MB/s peak
- Outgoing updates (fanout): 2.9GB/s
```

---

## 3. 🏗️ High-Level Architecture (8 min)

### 🌟 Key Design Principles:

* **Separation of concerns**: lifecycle, bidding, payments, search
* **Real-time core**: bid fanout, atomic updates
* **Consistency-first**: Redis + DB for authoritative state

### 🧱 Core Services:

```
Clients → [API Gateway] → Microservices:
         ├── Auth/User Service
         ├── Listing Service
         ├── Auction Lifecycle Service
         ├── Bidding Service ↔ Redis (live state)
         ├── Search Service ↔ Elasticsearch
         ├── Payment Service
         └── Notification Service

[WebSocket Gateway] ↔ Bidding Service

[Kafka] ↔ Async Workers:
         ├── Bid Notifier
         ├── Scheduler
         └── Escrow Processor
```

### Key APIs:

```http
POST /auctions/{id}/bids       # Place bid (WebSocket preferred)
GET  /auctions/{id}/bids       # Bid history
POST /payments/initiate        # Escrow after win
```

### WebSocket Event:

```json
// From client
{ "type": "place_bid", "auctionId": "abc", "amount": 120.00 }

// To watchers
{ "type": "bid_update", "auctionId": "abc", "price": 120.00, "bidder": "u123", "timeLeft": 25000 }
```

---

## 4. 🔎 Deep Dive (10 min)

### A. Real-Time Bidding: **Concurrency + Consistency**

#### ⟲ Flow:

1. Bid via WebSocket
2. Validate (auth, balance, auction open)
3. Atomically update Redis: `current_price`, `highest_bidder`
4. Persist to DB (async OK)
5. Broadcast update → watchers
6. Notify outbid users

#### ⚔️ Race Condition Handling:

* Redis Lua script for atomic price update
* Use timestamps + user ID for deterministic tie-breaking
* Optimistic locking fallback on DB write
