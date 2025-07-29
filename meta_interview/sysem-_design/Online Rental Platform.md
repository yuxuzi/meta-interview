# Airbnb-like Rental Platform System Design - 30-Minute Interview Prep

## 🎯 Interview Timeline (30 minutes)

- **Requirements Clarification (5 min)**: Understand scope and constraints
- **Capacity Estimation (3 min)**: Back-of-envelope calculations  
- **High-Level Design (8 min)**: Core architecture and APIs
- **Deep Dive (10 min)**: Focus on 1-2 critical components
- **Scaling & Wrap-up (4 min)**: Bottlenecks and solutions

---

## 1. Requirements Clarification (5 minutes)

### Key Questions to Ask the Interviewer:

- "What's the scale? How many users, properties, and bookings per day?"
- "Should we prioritize consistency or availability for bookings?"
- "What's the acceptable latency for search and booking updates?"
- "Do we need to handle concurrent booking attempts for the same dates?"
- "Should we support instant booking or require host approval?"

### Functional Requirements (Priority Order):

1. **Property listing management** (create, edit, photos, pricing, availability)
2. **Search and discovery** (location, dates, price, filters)
3. **Booking system** (reservation, calendar sync, payment)
4. **User management** (guests, hosts, profiles, authentication)
5. **Review and rating system** (guest ↔ host reviews)
6. **Payment processing** (booking fees, host payouts, refunds)
7. **Messaging system** (guest-host communication)
8. **Notifications** (booking confirmations, reminders)

### Non-Functional Requirements:

- **Scalability**: 50M users, 5M properties, 500K bookings/day
- **Low Latency**: <200ms for search, <100ms for booking confirmation
- **High Availability**: 99.9% uptime (bookings must be reliable)
- **Strong Consistency**: No double bookings for same dates
- **Data Integrity**: Accurate calendar availability
- **Security**: Secure payments, fraud prevention, data privacy

---

## 2. Capacity Estimation (3 minutes)

### 📊 Back-of-Envelope Calculations

#### Traffic Estimation:
```
Daily Active Users (DAU): 10M
Active properties: 5M total, 2M actively listed
Searches per user per day: 5 average
Total searches per day: 50M

Bookings per day: 500K
Average booking duration: 3 days
```

#### QPS Calculation:
```
Search QPS: 50M ÷ 86,400 = ~580 QPS
Peak search QPS: 580 × 5 = 2,900 QPS
Booking QPS: 500K ÷ 86,400 = ~6 QPS  
Peak booking QPS: 6 × 10 = 60 QPS
```

#### Storage Estimation:
```
Property data: 5KB per property × 5M = 25GB
User data: 2KB per user × 50M = 100GB
Booking data: 1KB per booking × 500K/day = 500MB/day
Photos: 10 photos × 2MB per property = 20MB × 5M = 100TB
Search indexes: ~50GB (Elasticsearch)

Annual growth:
- Booking history: 500MB × 365 = 180GB/year
- New photos: ~20TB/year
Total storage: ~120TB (mostly images in CDN)
```

#### Bandwidth Estimation:
```
Image delivery: 100TB ÷ 86,400s = ~1.2GB/s (CDN)
API requests: 2,900 QPS × 2KB = 5.8MB/s
Database writes: 60 QPS × 1KB = 60KB/s
```

---

## 3. High-Level Design (8 minutes)

### Core Services Architecture:

```
[Web/Mobile Apps] ←→ [CDN] ←→ [Load Balancer] ←→ [API Gateway]
                                                        ↓
                                               [Microservices]
                                               ├── User Service
                                               ├── Property Service  
                                               ├── Search Service
                                               ├── Booking Service
                                               ├── Payment Service
                                               ├── Review Service
                                               ├── Messaging Service
                                               └── Notification Service
                                                        ↓
                                            [Message Queue (Kafka)]
                                                        ↓
                                                [Async Workers]
                                                        ↓
                                               [Database Layer]
                                               ├── PostgreSQL (OLTP)
                                               ├── Redis (Cache)  
                                               ├── Elasticsearch (Search)
                                               └── S3 (Images/Files)
```

### Key APIs:

#### Authentication & Users:
```http
POST /api/v1/auth/login
GET /api/v1/users/{userId}
PUT /api/v1/users/{userId}/profile
```

#### Property Management:
```http
POST /api/v1/properties                    # Create listing
GET /api/v1/properties/{propertyId}        # Get property details  
PUT /api/v1/properties/{propertyId}        # Update listing
GET /api/v1/properties/{propertyId}/calendar # Get availability
PUT /api/v1/properties/{propertyId}/calendar # Update availability
```

#### Search & Discovery:
```http
GET /api/v1/search?location=NYC&checkin=2024-12-01&checkout=2024-12-03&guests=2
GET /api/v1/properties/{propertyId}/availability?start=2024-12-01&end=2024-12-31
```

#### Booking System:
```http
POST /api/v1/bookings                      # Create booking
GET /api/v1/bookings/{bookingId}          # Get booking details
PUT /api/v1/bookings/{bookingId}/confirm   # Confirm booking
DELETE /api/v1/bookings/{bookingId}       # Cancel booking
```

#### Payments:
```http
POST /api/v1/payments/reserve             # Reserve payment method
POST /api/v1/payments/charge              # Charge guest
POST /api/v1/payments/payout              # Pay host
```

### Database Design:

#### Properties (PostgreSQL):
```sql
CREATE TABLE properties (
    id UUID PRIMARY KEY,
    host_id UUID NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    address JSONB,
    property_type VARCHAR(50),
    max_guests INTEGER,
    bedrooms INTEGER,
    bathrooms INTEGER,
    amenities JSONB,
    base_price DECIMAL(10,2),
    cleaning_fee DECIMAL(10,2),
    instant_book BOOLEAN DEFAULT FALSE,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_properties_host ON properties(host_id);
CREATE INDEX idx_properties_location ON properties USING GIN(address);
CREATE INDEX idx_properties_type_guests ON properties(property_type, max_guests);
```

#### Bookings (PostgreSQL):
```sql
CREATE TABLE bookings (
    id UUID PRIMARY KEY,
    property_id UUID NOT NULL,
    guest_id UUID NOT NULL,
    host_id UUID NOT NULL,
    check_in_date DATE NOT NULL,
    check_out_date DATE NOT NULL,
    num_guests INTEGER NOT NULL,
    total_amount DECIMAL(10,2),
    booking_fee DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'pending',
    payment_status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_bookings_property_dates ON bookings(property_id, check_in_date, check_out_date)
WHERE status IN ('confirmed', 'checked_in');
CREATE INDEX idx_bookings_guest ON bookings(guest_id);
CREATE INDEX idx_bookings_host ON bookings(host_id);
```

#### Availability Calendar (PostgreSQL):
```sql
CREATE TABLE property_availability (
    property_id UUID NOT NULL,
    available_date DATE NOT NULL,
    is_available BOOLEAN DEFAULT TRUE,
    price DECIMAL(10,2),
    minimum_stay INTEGER DEFAULT 1,
    PRIMARY KEY (property_id, available_date)
);

CREATE INDEX idx_availability_date_range ON property_availability(available_date, is_available);
```

#### Search Index (Elasticsearch):
```json
{
  "properties": {
    "mappings": {
      "properties": {
        "id": {"type": "keyword"},
        "title": {"type": "text", "analyzer": "standard"},
        "description": {"type": "text"},
        "location": {"type": "geo_point"},
        "property_type": {"type": "keyword"},
        "max_guests": {"type": "integer"},
        "base_price": {"type": "float"},
        "amenities": {"type": "keyword"},
        "rating": {"type": "float"},
        "availability": {"type": "date_range"}
      }
    }
  }
}
```

---

## 4. Deep Dive Topics (10 minutes)

### A. Booking System & Concurrency Control

#### Booking Flow:
```
1. Guest searches available properties
2. Guest selects dates and initiates booking
3. System checks real-time availability
4. If available:
   - Lock dates temporarily (5 minutes)
   - Reserve payment method
   - Create pending booking
5. Process payment
6. If payment succeeds:
   - Confirm booking
   - Update calendar
   - Send notifications
7. If payment fails:
   - Release date lock
   - Clean up pending booking
```

#### Preventing Double Bookings:
```sql
-- Atomic availability check and booking creation
BEGIN;

-- Check availability with row-level locking
SELECT property_id FROM property_availability 
WHERE property_id = $1 
  AND available_date >= $2 AND available_date < $3
  AND is_available = true
FOR UPDATE;

-- If all dates available, create booking
INSERT INTO bookings (property_id, guest_id, check_in_date, check_out_date, status)
VALUES ($1, $2, $3, $4, 'confirmed');

-- Mark dates as unavailable
UPDATE property_availability 
SET is_available = false
WHERE property_id = $1 AND available_date >= $2 AND available_date < $3;

COMMIT;
```

#### Alternative: Optimistic Locking
```python
def create_booking(property_id, dates, guest_id):
    max_retries = 3
    for attempt in range(max_retries):
        try:
            # Check availability
            available = check_availability(property_id, dates)
            if not available:
                return {"error": "Dates not available"}
            
            # Create booking with version check
            booking = create_booking_record(property_id, dates, guest_id)
            
            # Update availability atomically
            rows_updated = update_availability(property_id, dates, expected_version)
            if rows_updated == 0:
                # Version mismatch, retry
                continue
                
            return {"booking_id": booking.id}
            
        except ConcurrencyError:
            if attempt == max_retries - 1:
                return {"error": "Unable to complete booking, please try again"}
            time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
```

### B. Search System Architecture

#### Search Flow:
```
1. User enters: location="San Francisco", dates="Dec 1-3", guests=2
2. API Gateway routes to Search Service
3. Search Service queries Elasticsearch:
   - Geographic search (location radius)
   - Date range availability filter  
   - Guest capacity filter
   - Price range, amenities filters
4. Get property IDs from search results
5. Fetch detailed property data from PostgreSQL
6. Apply business logic (pricing, availability)
7. Return ranked results
```

#### Elasticsearch Query:
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "geo_distance": {
            "distance": "10km",
            "location": {"lat": 37.7749, "lon": -122.4194}
          }
        },
        {"range": {"max_guests": {"gte": 2}}},
        {"range": {"base_price": {"gte": 50, "lte": 300}}}
      ],
      "filter": [
        {"term": {"status": "active"}},
        {"range": {"availability.start": {"lte": "2024-12-01"}}},
        {"range": {"availability.end": {"gte": "2024-12-03"}}}
      ]
    }
  },
  "sort": [
    {"_score": {"order": "desc"}},
    {"rating": {"order": "desc"}},
    {"base_price": {"order": "asc"}}
  ]
}
```

#### Search Optimization:
- **Caching**: Redis cache for popular searches (location + dates)
- **Pagination**: Limit results, implement cursor-based pagination
- **Auto-complete**: Separate service for location suggestions
- **Personalization**: ML ranking based on user history

### C. Payment Processing & Financial Consistency

#### Payment Flow (Escrow Model):
```
1. Guest booking confirmed → Charge guest's payment method
2. Hold funds in escrow account (not released to host yet)
3. Guest checks in → Funds eligible for release
4. Guest checks out + 24h grace period → Release to host
5. Handle disputes/refunds before final release
```

#### Saga Pattern for Distributed Transactions:
```python
class BookingSaga:
    def execute(self, booking_request):
        try:
            # Step 1: Reserve inventory
            reservation_id = self.inventory_service.reserve(
                property_id=booking_request.property_id,
                dates=booking_request.dates
            )
            
            # Step 2: Charge payment
            payment_id = self.payment_service.charge(
                amount=booking_request.total_amount,
                payment_method=booking_request.payment_method
            )
            
            # Step 3: Create booking record  
            booking_id = self.booking_service.create_booking(
                booking_request, reservation_id, payment_id
            )
            
            # Step 4: Send confirmation
            self.notification_service.send_confirmation(booking_id)
            
            return {"booking_id": booking_id}
            
        except Exception as e:
            # Compensating transactions
            self.rollback(reservation_id, payment_id)
            raise
    
    def rollback(self, reservation_id, payment_id):
        if payment_id:
            self.payment_service.refund(payment_id)
        if reservation_id:
            self.inventory_service.release_reservation(reservation_id)
```

---

## 5. Scaling & Wrap-up (4 minutes)

### Bottlenecks & Solutions:

#### 1. Search Performance:
- **Problem**: Complex geo-spatial queries, high read load
- **Solution**: Elasticsearch cluster with replicas, Redis caching, CDN for static results

#### 2. Booking Concurrency:
- **Problem**: Double bookings during high demand
- **Solution**: Database locks, optimistic concurrency, booking queue for popular properties

#### 3. Image Storage & Delivery:
- **Problem**: 100TB+ of property images
- **Solution**: S3 + CloudFront CDN, image optimization, lazy loading

#### 4. Database Scaling:
- **Problem**: Write bottlenecks for bookings/availability
- **Solution**: Read replicas, database sharding by geography, CQRS pattern

### Technology Stack:
```
Frontend: React/Vue.js, iOS/Android apps
Backend: Java/Python microservices  
API Gateway: Kong, AWS API Gateway
Message Queue: Apache Kafka, RabbitMQ
Databases: PostgreSQL, Redis, Elasticsearch
Storage: AWS S3, CloudFront CDN
Payments: Stripe, PayPal, Adyen
Monitoring: Prometheus, Grafana, DataDog
```

### Architecture Evolution:
```
Phase 1: Monolith (0-100K users)
├── Single database, simple search

Phase 2: Service-Oriented (100K-1M users)  
├── Separate search service
├── Redis caching
├── CDN for images

Phase 3: Microservices (1M-10M users)
├── Full service decomposition
├── Event-driven architecture
├── Database per service

Phase 4: Scale (10M+ users)
├── Multi-region deployment
├── Advanced caching strategies
├── ML-powered recommendations
```

---

## 🎯 Interview Tips:

1. **Start with core flow**: Search → View → Book → Pay → Stay
2. **Emphasize data consistency**: Booking conflicts are critical business issues
3. **Consider geography**: Multi-region deployment for global service
4. **Address peak loads**: Holiday seasons, popular events
5. **Security focus**: Payment processing, user data protection
6. **Mobile-first**: Most bookings happen on mobile devices

## 🔍 Common Follow-up Questions:

- "How do you handle cancellations and refunds?"
- "What's your strategy for handling peak demand (New Year's Eve)?"
- "How do you prevent fake listings or fraudulent bookings?"
- "How would you implement dynamic pricing?"
- "What happens if payment fails after booking confirmation?"
- "How do you handle different time zones for bookings?"
- "How would you implement a messaging system between guests and hosts?"

**Remember**: Rental platforms are fundamentally about **inventory management, payments, and trust** - demonstrate how your system ensures availability accuracy, financial integrity, and user safety!