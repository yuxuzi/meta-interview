# 🎯 Ride Hailing System Design - Interview Conversation Script

*Practice this conversational flow to nail your Meta interview*

---

## **INTERVIEWER:** "Design a global, low-latency ride-hailing platform like Uber. I want to see how you handle real-time matching, location tracking, and scale."

**YOU:** "Great! Let me start by clarifying the requirements and scope. I'm thinking of a system that connects riders with drivers in real-time globally. Let me break this down..."

---

## 1️⃣ **FUNCTIONAL REQUIREMENTS** *(2 min)*

**YOU:** "So we need two main user flows. For riders: they request a ride, get matched with a driver, track the driver's arrival, take the trip, and pay. For drivers: they go online, receive trip requests via real-time notifications, accept or decline, navigate to pickup, complete trips, and get paid.

The core technical challenges are real-time matching under 3 seconds, live location tracking with sub-200ms updates, dynamic surge pricing, and payment processing. 

For scale, I'm assuming 10 million daily active users, 1 million concurrent drivers, about 5 million trips per day, and we need 99.99% uptime globally. Does this sound right?"

**INTERVIEWER:** "Yes, that scope works. How would you estimate capacity?"

---

## 2️⃣ **CAPACITY ESTIMATION** *(3 min)*

**YOU:** "Let me do some quick math. With 10 million daily users and 5 million trips, that's 0.5 trips per user on average. Peak traffic is usually 3x average, so around 175 trips per second at peak.

For location updates, 1 million drivers updating every 10 seconds gives us 100,000 location updates per second. During matching, we might have about 1,300 concurrent trip requests being processed at any time.

Storage-wise, each trip is maybe 2KB of data, so 5 million trips is 10GB per day. Location data is much heavier - 100K updates per second at 200 bytes each gives us about 500GB per day after compression.

For memory, we need to keep live driver locations cached. That's about 1 million drivers at 200 bytes each, so roughly 200MB per region. Very manageable in Redis."

**INTERVIEWER:** "Good estimates. Show me the high-level architecture."

---

## 3️⃣ **HIGH-LEVEL DESIGN** *(8 min)*

**YOU:** "I'm designing this as a microservices architecture. Mobile apps hit a CDN and load balancer, then go through an API Gateway.

The core services are:
- User Service handling auth and profiles
- Trip Service managing trip lifecycle and state
- Location Service for GPS ingestion and real-time updates  
- Matching Service for connecting riders with drivers
- Payment Service for billing and payouts
- Notification Service for push notifications
- Pricing Service for surge calculation
- ETA/Map Service for routing and navigation

For data storage, I'm using PostgreSQL for transactional data like trips and users, Redis for live data like driver locations and caching, Cassandra for location history, and Kafka as the event backbone.

The key APIs are REST endpoints for trip management, location updates, and WebSocket connections for real-time communication. Drivers maintain persistent WebSocket connections to receive instant trip offers, and riders connect during trips for live updates."

**INTERVIEWER:** "Interesting. How exactly does the real-time matching work? That seems like the most critical component."

---

## 4️⃣ **DEEP DIVE - REAL-TIME MATCHING** *(10 min)*

**YOU:** "This is where it gets interesting. I'm using geographic partitioning with Uber's H3 hexagonal grid system. Instead of naive latitude/longitude, H3 gives us uniform hexagonal cells about 460 meters in diameter. This eliminates edge cases and makes neighbor traversal efficient.

Here's the matching flow: When a rider requests a trip, we map their pickup location to an H3 cell. The matching service queries Redis for available drivers in that cell using geospatial commands. We score drivers based on distance, ETA, rating, and acceptance history.

Now here's the key part - instead of polling, all drivers maintain WebSocket connections to our matching service. We send trip offers simultaneously to the top 3 scored drivers via WebSocket. Each offer has a 30-second expiry timer.

Drivers respond in real-time - accept or decline through the WebSocket. First acceptance wins, and we immediately notify the other drivers that the trip is taken, plus instantly notify the rider that we found their driver.

To prevent thrashing, we implement several mechanisms: drivers who decline get a brief soft-lock, we track acceptance rates to adjust scoring, and we have circuit breakers if drivers are unresponsive."

**INTERVIEWER:** "What about the location tracking pipeline?"

**YOU:** "The location pipeline has two paths. The hot path handles real-time updates: driver apps send GPS coordinates via WebSocket, our location gateway does map-matching and applies Kalman filtering to smooth out GPS noise, then publishes to Kafka.

Location processors consume from Kafka and do two things: update Redis with the driver's current position for matching queries, and fan out the location to any connected riders tracking that driver.

The cold path asynchronously writes location history to Cassandra for analytics and ML training, with a 90-day TTL for privacy."

**INTERVIEWER:** "How do you handle surge pricing?"

**YOU:** "Surge pricing runs per H3 cell in real-time. We monitor the demand-to-supply ratio - active trip requests divided by available drivers in each cell. 

An ML model considers this ratio plus contextual features like time of day, weather, local events, and historical patterns. We apply EMA smoothing over 5-minute windows to prevent price shock, and use neighbor diffusion so adjacent cells influence each other to prevent arbitrage.

Prices lock when a rider requests, valid for 2 minutes, so they don't get surprised by changes during matching."

**INTERVIEWER:** "What about payments and consistency?"

**YOU:** "Payments need strong consistency. When a trip completes, we use the outbox pattern - we update the trip status and write a payment event to an outbox table in the same database transaction.

A separate payment processor reads the outbox and handles the actual charging through external payment providers like Stripe. We use idempotency keys to prevent double-charging if there are retries.

We maintain an internal ledger for all financial transactions - rider charges, driver payouts, platform fees - for reconciliation and audit purposes."

---

## 5️⃣ **SCALING UP** *(5 min)*

**INTERVIEWER:** "How does this scale globally and handle failures?"

**YOU:** "For horizontal scaling, each service scales independently. The matching service scales by H3 cell coverage and WebSocket connection count. We can handle about 10,000 WebSocket connections per instance at roughly 10MB memory.

For databases, we shard PostgreSQL by user_id with read replicas in each region. Redis is geo-sharded by H3 cell using consistent hashing. Cassandra naturally partitions by driver_id and date.

Globally, we deploy regional clusters with data residency compliance. Each region handles its own matching and payments independently, but user profiles and driver verification sync cross-region.

For failures, we have multiple layers of resilience. Circuit breakers protect external services. If the ML pricing service fails, we fall back to simple supply-demand ratios. If Redis goes down, we can query PostgreSQL with spatial indexes, though with higher latency.

WebSocket connections auto-reconnect with exponential backoff. If a regional matching service fails, we can temporarily route to neighboring regions.

The key SLIs we monitor are trip matching time P95 under 3 seconds, location update fanout P95 under 150ms, and overall API response P99 under 300ms."

**INTERVIEWER:** "What are the main trade-offs in your design?"

**YOU:** "The biggest trade-off is consistency versus performance. We chose strong consistency for payments and trip state because money is involved, but eventual consistency for locations and surge pricing because real-time performance matters more.

Using WebSockets gives us great real-time performance but adds complexity in connection management and scaling. We could use HTTP polling but that would never hit our 3-second matching SLA.

H3 geographic partitioning is more complex than simple geohashing but gives us much better load distribution and eliminates edge cases around map boundaries.

The event-driven architecture with Kafka adds complexity but gives us audit trails and the ability to replay events, which is crucial for debugging production issues in a financial system."

**INTERVIEWER:** "How would you extend this for autonomous vehicles?"

**YOU:** "Great question! For autonomous vehicles, the main changes would be in the matching algorithm - AVs could optimize for longer trips since there's no driver fatigue, and we'd need predictive positioning to pre-stage vehicles in high-demand areas.

We'd also need enhanced location tracking with sensor fusion beyond just GPS, and integration with traffic management systems for coordinated routing. The core architecture would remain the same though."

---

## 🏆 **CLOSING SUMMARY**

**YOU:** YOU: "To summarize, this design handles 10 million users globally through H3 geographic partitioning, WebSocket-based real-time matching under 3 seconds, and Kafka event streams for reliable service coordination and location fanout. The system gracefully degrades during failures and scales horizontally across all components."
Key points packed in:

✅ H3 geographic partitioning - core scaling strategy
✅ WebSocket real-time matching - performance differentiator
✅ Kafka event streams - reliability & coordination backbone
✅ Graceful degradation - production readiness
✅ Horizontal scaling - growth capability

Clean, concise, and hits all the technical highlights"

**INTERVIEWER:** "Excellent. You've shown strong understanding of distributed systems, real-time processing, and production considerations."

---

## 💡 **KEY MEMORIZATION POINTS**

### **Numbers to Remember:**
- 10M DAU, 1M drivers, 5M trips/day, 175 peak TPS
- 100K location updates/sec, <3s matching, <150ms fanout
- H3 resolution 8 = 460m diameter cells

### **Tech Stack:**
- **Real-time:** WebSocket + Kafka + Redis
- **Storage:** PostgreSQL + Cassandra + Redis
- **Geographic:** H3 hexagonal grid partitioning
- **Patterns:** Outbox, Event Sourcing, Circuit Breaker

### **Key Flows:**
1. **Matching:** H3 cell → Redis query → Score → WebSocket fanout → First accept wins
2. **Location:** WebSocket → Kafka → Redis + fanout → Cassandra history
3. **Payment:** Trip complete → Outbox → PSP + Ledger → Driver payout

### **Failure Handling:**
- Circuit breakers for external services
- Graceful degradation (Redis fails → PostgreSQL spatial)
- WebSocket auto-reconnect
- Regional failover