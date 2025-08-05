# Uber-like Ride Hailing System Design — 30-Minute Interview Guide

## 1️⃣ REQUIREMENTS CLARIFICATION (3-4 minutes)

### 🎯 Core Objective
Design a scalable ride-hailing platform that connects riders with drivers in real-time.

### ✅ Functional Requirements
- **Rider**: Request rides, track driver, make payments, rate trips
- **Driver**: Accept/decline rides, navigate, receive payments
- **Real-time Matching**: Connect nearby drivers with riders
- **Location Services**: GPS tracking, ETA calculations
- **Payment**: Multiple methods, fare calculation, driver payouts
- **Surge Pricing**: Dynamic pricing based on demand/supply

### 🔒 Non-Functional Requirements
- **Scale**: 10M daily active users, 1M active drivers
- **Performance**: <3s ride matching, <100ms location updates
- **Availability**: 99.99% uptime
- **Global**: Multi-city support with regional data centers

---

## 2️⃣ CAPACITY ESTIMATION (4-5 minutes)

### 📊 Key Metrics

| Metric | Estimate | Notes |
|--------|----------|-------|
| **Daily Trips** | 5M | 0.5 trips per active user |
| **Peak Trips/Second** | 175 | 3x peak multiplier |
| **Location Updates/Second** | 100K | 1M drivers × 0.1/sec |
| **Daily Storage** | ~1TB | Location + trip data |
| **Annual Storage** | ~300TB | With compression |

---

## 3️⃣ HIGH-LEVEL ARCHITECTURE (8-10 minutes)

### 🏗️ System Components

```
[Mobile Apps] → [Load Balancer] → [API Gateway]
                                       ↓
┌─────────────────┬─────────────────┬─────────────────┐
│  User Service   │Location Service │ Matching Service│
│  Trip Service   │Payment Service  │Notification Svc │
└─────────────────┴─────────────────┴─────────────────┘
                        ↓
┌─────────────────┬─────────────────┬─────────────────┐
│ Kafka (Events)  │Redis (Cache/    │ PostgreSQL      │
│                 │ Real-time Data) │ (Persistent)    │
└─────────────────┴─────────────────┴─────────────────┘
```

### 🌐 Core APIs

```http
# Trip Management
POST /trips/request           # Request ride
PUT  /trips/{id}/accept       # Driver accepts
PUT  /trips/{id}/start        # Trip starts
PUT  /trips/{id}/complete     # Trip ends

# Real-time Updates (WebSocket)
/ws/driver/location          # Driver location stream  
/ws/rider/trip-updates       # Trip status updates
```

### 🗄️ Key Data Models

```sql
-- Core trip table
CREATE TABLE trips (
  id UUID PRIMARY KEY,
  rider_id UUID,
  driver_id UUID,
  pickup_lat DECIMAL(10,8),
  pickup_lng DECIMAL(11,8), 
  destination_lat DECIMAL(10,8),
  destination_lng DECIMAL(11,8),
  status ENUM('requested','matched','in_progress','completed'),
  fare_amount DECIMAL(10,2),
  surge_multiplier DECIMAL(3,2) DEFAULT 1.00,
  created_at TIMESTAMP
);

-- Driver location (Redis + Cassandra for history)
driver_locations: {
  driver_id: "123",
  lat: 37.7749,
  lng: -122.4194,
  timestamp: 1690123456789,
  status: "available"
}
```

---

## 4️⃣ DETAILED DESIGN DEEP DIVES (8-10 minutes)

### 🎯 Real-time Driver Matching

**Matching Flow**:
```
1. Rider requests → Find nearby drivers (Redis GeoSpatial)
2. Apply filters (rating, vehicle type)
3. Send to top 3 drivers simultaneously  
4. First acceptance wins → Notify all parties
```

**Geospatial Queries**:
```python
# Redis commands for driver matching
GEOADD drivers:online -122.4194 37.7749 "driver_123"
GEORADIUS drivers:online -122.4194 37.7749 5 km WITHDIST

# Smart matching with ML optimization
class MatchingService:
    def find_optimal_driver(self, trip_request):
        nearby_drivers = self.geo_query(trip_request.pickup)
        scored_drivers = self.ml_model.score_drivers(
            drivers=nearby_drivers,
            trip=trip_request,
            traffic_data=self.get_traffic_data()
        )
        return scored_drivers[:3]  # Top 3 candidates
```

### 🗺️ Location Management

**Real-time Pipeline**:
```
Driver App → WebSocket → Location Service → Redis Pub/Sub → Matching Service
                                ↓
                        Cassandra (Historical)
```

**Optimization Strategies**:
- **Adaptive frequency**: 4s driving, 30s stationary
- **Geofencing**: Higher frequency in demand hotspots  
- **Battery optimization**: Reduce precision when not matching

### 💰 Dynamic Pricing

**Surge Algorithm**:
```python
def calculate_surge(location, time):
    demand = count_active_requests(location)
    supply = count_available_drivers(location) 
    ratio = demand / max(supply, 1)
    
    # Apply smoothing + caps
    surge = min(ratio * 0.5, 3.0)
    return smooth_price_changes(surge)
```

### 💳 Payment Processing

**Fault-tolerant Flow**:
```python
def process_payment(trip_id, amount):
    # Idempotent processing
    if payment_exists(trip_id):
        return existing_payment(trip_id)
    
    try:
        result = payment_gateway.charge(amount)
        if result.success:
            record_payment(trip_id, result)
            queue_driver_payout(trip_id)
        return result
    except Exception:
        queue_retry(trip_id, exponential_backoff=True)
```

---

## 5️⃣ SCALING & PERFORMANCE (3-4 minutes)

### 🎯 Key Optimizations

| Challenge | Solution | Impact |
|-----------|----------|---------|
| **Location Load** | Redis sharding + pub/sub | 100K updates/sec |
| **DB Hotspots** | Geographic sharding | Even load distribution |
| **Matching Speed** | In-memory indexes + caching | <3s matching |
| **Global Latency** | Regional DCs + CDN | <100ms API calls |

### 🔧 Scalability Patterns

- **Microservices**: Independent scaling per service
- **Database Sharding**: By city/region + read replicas  
- **Caching**: Multi-layer (Redis, CDN, app cache)
- **Auto-scaling**: Based on traffic patterns and demand

---

## 6️⃣ MONITORING & RELIABILITY (2-3 minutes)

### 📊 Key Metrics
- **Business**: Trips/min, driver utilization, surge effectiveness
- **Technical**: API latency P95, error rates, cache hit ratio
- **Safety**: Emergency usage, driver verification status

### 🛡️ Fault Tolerance
- **Circuit breakers** for external services
- **Database replication** across AZs
- **Graceful degradation** during outages
- **Disaster recovery** with cross-region backups

---

## 7️⃣ INTERVIEW TIPS & TRADE-OFFS (1-2 minutes)

### ✅ Key Discussion Points
1. **Consistency**: Strong for payments, eventual for locations
2. **Real-time vs Batch**: Location updates vs analytics processing  
3. **Cost vs Performance**: Balance infrastructure costs with UX

### 🎯 Advanced Concepts to Mention
- **Geospatial indexing** (R-trees, QuadTrees)
- **Event sourcing** for trip state management
- **Machine learning** for demand prediction & ETA
- **Chaos engineering** for reliability testing

### 🚀 Follow-up Extensions
- **Multi-modal transport** (bikes, scooters, transit)
- **Autonomous vehicle** integration
- **Safety features** (real-time monitoring, SOS)
- **Carbon tracking** and offset programs

---

## 🏆 WINNING SUMMARY

This design handles **10M users** with:
- ⚡ **Real-time matching** in <3 seconds using geospatial indexing
- 🌍 **Global scale** with regional sharding and data centers  
- 💰 **Smart pricing** with ML-driven surge calculations
- 🛡️ **High reliability** through fault-tolerant architecture
- 📊 **Production-ready** with comprehensive monitoring

Shows mastery of **distributed systems**, **real-time processing**, and **mobile-scale architecture**.