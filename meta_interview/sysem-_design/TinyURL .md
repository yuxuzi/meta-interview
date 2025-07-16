# 📝 Final Architecture Summary

# TinyURL System Design - Interview Guide

## 🎯 Interview Timeline (30 minutes)

* **Requirements** (5 min): Functional and non-functional requirements
* **Capacity Estimation** (5 min): Quick back-of-envelope calculations
* **High-Level Design** (15 min): Core architecture and APIs
* **Deep Dive** (5 min): URL encoding algorithm

## 1. Requirements (5 minutes)

### Functional Requirements

* Shorten long URLs to short URLs
* Redirect short URLs to original URLs
* Custom aliases (optional)
* Basic analytics (click count)

### Non-Functional Requirements

* **Scale** : 100M URLs/day, 10B clicks/day
* **Read/Write Ratio** : 100:1 (read-heavy)
* **Latency** : <100ms for redirects
* **Availability** : 99.9%

## 2. Capacity Estimation (5 minutes)

### Traffic

* **Writes** : 100M URLs/day = ~1,200 QPS
* **Reads** : 10B clicks/day = ~120K QPS
* **Peak Load** : 2x average = 2,400 writes/sec, 240K reads/sec

### Storage

* **URL Record** : ~500 bytes (URLs + metadata)
* **Daily Storage** : 100M × 500 bytes = 50GB/day
* **5-Year Storage** : ~90TB

### Cache

* **Hot Data** : 20% of URLs = 18TB
* **Active Cache** : 80% hit rate = ~4TB memory needed

## 3. High-Level Architecture (15 minutes)

```
Users
  ↓
API Gateway (Authentication, Rate Limiting)
  ↓
Load Balancer
  ↓
┌─────────────────┬──────────────────┬────────────────────┐
│ URL Generation  │ Redirect Service │ User Management    │
│ Service         │ (Multi-region)   │ Service            │
└─────────────────┴──────────────────┴────────────────────┘
  ↓                 ↓                  ↓
┌─────────────────┬──────────────────┬────────────────────┐
│ PostgreSQL      │ Redis Cache      │ User Database      │
│ (Master)        │ (Distributed)    │                    │
└─────────────────┴──────────────────┴────────────────────┘
  ↓
PostgreSQL Read Replicas
```

### Core Services

1. **URL Generation Service** : Creates short URLs
2. **Redirect Service** : Handles URL redirects (read-heavy)
3. **User Management Service** : Handles user accounts and custom aliases

### Key APIs

#### Shorten URL

http

```http
POST /api/shorten
{
  "original_url": "https://example.com/very/long/url",
  "custom_alias": "my-link",  // optional
  "user_id": 12345           // optional
}

Response:
{
  "short_url": "https://tiny.ly/abc123",
  "original_url": "https://example.com/very/long/url"
}
```

#### Redirect URL

http

```http
GET /abc123
Response:301 Redirect
Location:https://example.com/very/long/url
```

### Database Schema

sql

```sql
-- URLs table
CREATETABLE urls (
    id BIGSERIAL PRIMARYKEY,
    short_url VARCHAR(7)UNIQUENOTNULL,
    original_url TEXTNOTNULL,
    user_id BIGINT,
    click_count BIGINTDEFAULT0,
    created_at TIMESTAMPDEFAULTNOW(),
INDEX idx_short_url (short_url)
);
```

## 4. Deep Dive: URL Encoding (5 minutes)

### Base62 Encoding Algorithm

python

```python
classURLEncoder:
def__init__(self):
        self.chars ="0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
        self.counter =0# Global counter
  
defencode(self, number):
	if number ==0:
		return self.chars[0]
  
        result =""
	while number >0:
            result = self.chars[number %62]+ result
            number //=62
	return result
  
def generate_short_url(self):
        self.counter +=1
	return self.encode(self.counter)
```

### Why Base62?

* **URL-safe** : Only alphanumeric characters
* **Compact** : 7 characters = 62^7 = 3.5 trillion URLs
* **No collisions** : Counter-based generation
* **Readable** : No special characters

## 5. Scaling Considerations

### Database Scaling

* **Read Replicas** : Handle 100:1 read/write ratio
* **Sharding** : Shard by short_url hash if needed
* **Connection Pooling** : Use PgBouncer

### Caching Strategy

* **Redis Cache** : Cache hot URLs (80% hit rate)
* **Cache TTL** : 1 hour for URL mappings
* **Cache-aside pattern** : Check cache first, then database

## Interview Tips

### What to Emphasize

* "The key insight is the 100:1 read/write ratio"
* "Base62 encoding gives us collision-free short URLs"
* "Redis cache is critical for redirect performance"
* "Read replicas handle the read-heavy workload"

### Common Follow-up Questions

* **Q** : "How do you handle custom aliases?"
* **A** : "Check if alias exists, if not, use it directly instead of generated ID"
* **Q** : "What about URL expiration?"
* **A** : "Add expires_at column, check during redirect, cleanup with background job"
* **Q** : "How do you prevent spam?"
* **A** : "Rate limiting at API Gateway, user authentication for custom aliases"

## Key Design Decisions

1. **Counter-based encoding** : Simple, collision-free
2. **Read replicas** : Handle read-heavy traffic
3. **Redis caching** : 80% cache hit rate for performance
4. **Simple analytics** : Just click counts, no complex event processing
5. **Three-service architecture** : URL generation, redirect, user managementData Layer
