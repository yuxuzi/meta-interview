# 🚀 Elite Ride-Hailing System Design - Meta 30min Interview

## 1️⃣ FUNCTIONAL REQUIREMENTS (2 min)

### Core Features
- **Rider Flow**: Request ride → Get matched → Track driver → Pay → Rate
- **Driver Flow**: Go online → Receive requests → Accept/Decline → Navigate → Get paid
- **Real-time Matching**: Connect nearby available drivers with riders <3s
- **Location Tracking**: Live GPS updates, ETA calculations, route optimization
- **Dynamic Pricing**: Surge pricing based on supply/demand
- **Payment Processing**: Multiple payment methods, automatic billing, driver payouts

### Scale Targets
- 10M Daily Active Users, 1M concurrent drivers
- 5M trips/day, peak 175 trips/second
- Global deployment across 100+ cities
- 99.99% uptime, <3s matching SLA

---

## 2️⃣ CAPACITY ESTIMATION (3 min)

### Traffic Analysis
```
Daily Active Users: 10M
Active Drivers: 1M concurrent
Daily Trips: 5M (0.5 trips per user)
Peak TPS: 175 trips/second
Location Updates: 100K/second (1M drivers × 0.1 Hz)
Concurrent Matches: ~1,300 active auctions
```

### Storage Requirements
```
Trip Data: 5M trips × 2KB = 10GB/day → 3.6TB/year
Location Data: 100K updates/sec × 200B × 86400s = 1.7TB/day (compressed: 500GB)
User/Driver Profiles: 11M users × 5KB = 55GB
Payment Records: 5M × 500B = 2.5GB/day
Total: ~2TB/day, ~700TB/year
```

### Memory & Compute
```
Redis (Hot Data): 1M drivers × 200B = 200MB per region
Matching Service: 1,300 auctions × 10KB state = 13MB active memory
Location Cache: 100K updates buffered = 20MB
Total Memory: <1GB per service instance
```

---

## 3️⃣ HIGH-LEVEL DESIGN (8 min)

### System Architecture
```
Mobile Apps → CDN/Load Balancer → API Gateway
                                     ↓
┌─────────────┬──────────────┬────────────────┬──────────────┐
│ User Service│ Trip Service │ Location Service│ Match Service│
│             │              │                │              │
│ • Auth      │ • Trip CRUD  │ • GPS Ingest   │ • Driver     │
│ • Profile   │ • State Mgmt │ • Real-time    │   Discovery  │
│ • Rating    │ • History    │   Updates      │ • Auction    │
└─────────────┴──────────────┴────────────────┴──────────────┘
                              ↓
┌─────────────┬──────────────┬────────────────┬──────────────┐
│Payment      │Notification  │ Pricing Service│ ETA/Map Svc  │
│Service      │Service       │                │              │
│             │              │ • Surge Calc   │ • Route Calc │
│• Billing    │• Push/SMS    │ • Dynamic      │ • Traffic    │
│• Payouts    │• WebSocket   │   Pricing      │ • Navigation │
└─────────────┴──────────────┴────────────────┴──────────────┘
                              ↓
┌─────────────┬──────────────┬────────────────┬──────────────┐
│   Kafka     │    Redis     │  PostgreSQL    │  Cassandra   │
│             │              │                │              │
│• Event Bus  │• Live Data   │• Trip Records  │• Location    │
│• Commands   │• Driver Sets │• User Data     │  History     │
│• Location   │• Geospatial  │• Payments      │• Analytics   │
│  Stream     │  Queries     │• Ledger        │              │
└─────────────┴──────────────┴────────────────┴──────────────┘
```

### Key APIs
```
POST   /trips/request           # Request ride
PUT    /trips/{id}/start        # Trip starts  
PUT    /trips/{id}/complete     # Trip completes
GET    /trips/{id}/status       # Real-time status

POST   /location/update         # Driver location
GET    /location/nearby         # Find nearby drivers

WebSocket /driver/connect       # Driver real-time connection
WebSocket /rider/connect        # Rider real-time updates
  - trip_offers (to drivers)
  - accept_trip/decline_trip (from drivers)  
  - driver_found/driver_arriving (to riders)
  - location_updates (bidirectional)
```

### Data Models
```
Trip: {
  id, rider_id, driver_id,
  pickup_location {lat, lng, address},
  destination {lat, lng, address},
  status: [requested, matched, started, completed, cancelled],
  fare_amount, surge_multiplier,
  created_at, started_at, completed_at
}

Driver: {
  id, location {lat, lng, h3_cell}, 
  status: [offline, available, busy],
  vehicle_info, rating, last_seen
}

Location_Update: {
  driver_id, lat, lng, heading, speed,
  timestamp, h3_cell
}
```

---

## 4️⃣ DEEP DIVE (10 min)

### A) Geographic Partitioning Strategy
**H3 Hexagonal Grid System**:
- Resolution 8: ~460m diameter cells (city block level)
- Natural load distribution, no polar distortion
- Efficient neighbor traversal for spillover scenarios
- Partition matching actors by H3 cell for locality

**Benefits over naive lat/lng**:
- Predictable hotspots enable better caching
- Hierarchical structure supports multi-resolution queries  
- Uniform cell shapes prevent edge case handling

### B) WebSocket-Based Real-Time Matching
**Persistent Connection Architecture**:
- All drivers maintain WebSocket connections to matching service
- Riders connect via WebSocket during trip request flow
- Connection pooling: ~10K connections per matching service instance
- Regional WebSocket gateways for low latency

**Real-Time Matching Flow**:
1. **Trip Request**: Rider submits via REST API
2. **Driver Discovery**: Query Redis geospatial index for H3 cell
3. **Scoring & Selection**: Rank drivers by distance, rating, acceptance rate
4. **WebSocket Broadcast**: Send trip offers to top 3 drivers simultaneously
5. **Real-Time Responses**: Drivers respond accept/decline via WebSocket
6. **Winner Notification**: First acceptance wins, instant notification to rider

**WebSocket Message Types**:
```
// To Driver
{type: "TRIP_OFFER", trip_id, pickup, destination, fare, expires_in: 30s}
{type: "TRIP_CANCELLED", trip_id}

// From Driver  
{type: "ACCEPT_TRIP", trip_id, driver_id}
{type: "DECLINE_TRIP", trip_id, driver_id, reason}

// To Rider
{type: "DRIVER_FOUND", driver_info, eta}
{type: "DRIVER_ARRIVING", current_location, eta}
```

**Anti-Thrashing via WebSocket**:
- Real-time driver status updates prevent stale offers
- Immediate offer cancellation when driver goes offline
- Dynamic timeout adjustment based on response patterns
- Connection health monitoring with auto-reconnect

### C) Location Processing Pipeline
**Hot Path (Real-time)**:
```
Driver App → WebSocket → Location Gateway → Kafka Topic → 
Cell Processor → Redis UPDATE + Rider WebSocket Fanout
```

**Optimizations**:
- **Map Matching**: Snap GPS to road network, reduce noise
- **Kalman Filtering**: Smooth erratic GPS signals
- **Adaptive Frequency**: 4s when moving, 30s when stationary
- **Batch Processing**: 50ms micro-batches for efficiency

**Cold Path (Historical)**:
- Async append to Cassandra with 90-day TTL
- Downsample for analytics (privacy-preserving aggregation)
- S3 archival for ML model training

### D) Dynamic Surge Pricing
**Real-time Calculation**:
- Monitor demand/supply ratio per H3 cell
- ML model considers: time, weather, events, historical patterns
- EMA smoothing prevents price shock (5-minute windows)
- Neighbor diffusion prevents arbitrage opportunities

**Price Locking**:
- Quote locked for 2 minutes at request time
- Hedging buffer for price movements during matching
- Driver compensation adjustment if surge drops significantly

### E) Payment Architecture
**Strong Consistency Model**:
- Outbox pattern: Trip completion + Payment event in same transaction
- Idempotency keys prevent double-charging
- Dual-write: External PSP + Internal ledger for reconciliation

**Fault Tolerance**:
- Async retry with exponential backoff
- Manual reconciliation dashboard for edge cases
- Allow trip completion even if payment temporarily fails

---

## 5️⃣ SCALE UP (5 min)

### Horizontal Scaling Strategies

**Service-Level Scaling**:
- **Matching Service**: Scale by WebSocket connection count + H3 cell coverage
- **WebSocket Gateway**: Horizontal scaling with connection load balancing
- **Location Service**: Scale by update rate, stateless processing  
- **Trip Service**: Scale by active trip count, event-sourced
- **Payment Service**: Scale by transaction volume, async processing

**WebSocket Scaling Considerations**:
- **Connection Distribution**: Sticky sessions or consistent hashing by driver_id
- **Message Routing**: Kafka topics for cross-instance message delivery  
- **Connection Recovery**: Auto-reconnect with exponential backoff
- **Memory Management**: ~1KB per connection × 10K connections = 10MB per instance

**Database Scaling**:
- **PostgreSQL**: Shard by user_id + read replicas per region
- **Redis**: Geo-sharded clusters, consistent hashing by H3 cell
- **Cassandra**: Partition by (driver_id, date) for location history

### Performance Optimizations

**Caching Strategy**:
```
L1: Application cache (driver scores, ETA estimates)
L2: Redis cluster (live driver locations, surge pricing)  
L3: CDN (static content, map tiles)
```

**Connection Optimization**:
- **WebSocket Pooling**: Persistent connections with heartbeat (30s)
- **Message Batching**: Bundle location updates in 100ms windows
- **HTTP/2 Multiplexing**: For REST API calls
- **Redis Pipelining**: For batch geospatial operations
- **Connection Pre-warming**: Scale WebSocket gateways before peak hours

### Global Distribution

**Regional Architecture**:
- Each region: independent matching + payments
- Cross-region: user profiles, driver verification
- Edge locations: location ingestion, map services
- Circuit breakers for cross-region fallback

**Data Residency**:
- Trip data localized per country/region
- Location data with configurable retention
- Payment data compliance (PCI-DSS, GDPR)

### Monitoring & Alerting

**Key SLIs**:
- Trip matching time P95 < 3s
- Location update fanout P95 < 150ms  
- API response time P99 < 300ms
- Driver utilization rate > 70%

**Failure Detection**:
- Health checks every 30s with exponential backoff
- Circuit breaker pattern for external dependencies
- Canary deployments with automatic rollback
- Real-time anomaly detection on business metrics

**Capacity Planning**:
- Auto-scaling based on pending trip queue depth
- Predictive scaling for known traffic patterns
- Resource pooling across services during off-peak
- Cost optimization through spot instances for batch processing

---

## 🏆 ARCHITECTURE HIGHLIGHTS

✅ **Geographic partitioning** with H3 cells - scales naturally with city growth  
✅ **Cell-local actors** - bounded latency guarantees for matching  
✅ **Event-driven architecture** - strong consistency where needed, eventual elsewhere  
✅ **Multi-layer caching** - <150ms location updates despite mobile network variance  
✅ **Graceful degradation** - system remains functional during partial failures  
✅ **Global scale** - regional autonomy with cross-region data synchronization

**Production-ready for 10M+ users with sub-3-second matching worldwide**