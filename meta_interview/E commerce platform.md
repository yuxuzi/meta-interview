# Amazon-like E-commerce Platform Design — 30-Minute Interview Guide

## 1️⃣ REQUIREMENTS CLARIFICATION (3-4 minutes)

### 🎯 Core Objective

Design a scalable e-commerce platform like Amazon.

### ✅ Functional Requirements

- Product catalog
- Inventory management
- Shopping cart and order management
- Secure payments
- Seller portal
- Product reviews and ratings
- User management
- Notification service (email/SMS)

### 🔒 Non-Functional Requirements

- **Scale**: 10K DAUs, 10K peak concurrent users
- **Performance**: <200ms latency
- **Availability**: 99.99% uptime
- **Reliability**: Consistent inventory, durable orders
- **Security**: Secure transactions, data privacy

---

## 2️⃣ CAPACITY ESTIMATION (4-5 minutes)

### 📊 Traffic & Storage Analysis

| Metric               | Estimate | Notes                    |
| -------------------- | -------- | ------------------------ |
| Daily Active Users   | 10,000   | Peak concurrency: 10,000 |
| Avg Queries/User/Day | 10       | 100K queries/day         |
| Avg Orders/User/Day  | 1        | 10K orders/day           |
| Avg Page Size        | 10KB     | Product info             |
| Daily Data Ingestion | ~1GB     | 1 year ≈ 360–500 GB    |

---

## 3️⃣ HIGH-LEVEL ARCHITECTURE (8-10 minutes)

### 🏗️ Architecture Diagram

```text
[Clients (Web/Mobile)]
        ↓
[Load Balancer]
        ↓
[API Gateway]
├── User Service
│   └── PostgreSQL (User DB)
├── Product Catalog Service
│   ├── Redis (Cache)
│   └── PostgreSQL (Catalog DB)
├── Inventory Service
│   ├── Redis (Stock Cache)
│   └── PostgreSQL (Inventory DB)
├── Order Service
│   ├── Kafka (Order Events)
│   └── PostgreSQL (Order DB)
├── Payment Service
│   └── External Payment Gateway
├── Notification Service
│   └── Email/SMS provider
```

### 🌐 REST API Examples

```http
# User
POST /users/register
POST /users/login
GET /users/profile

# Product Catalog
GET /products
GET /products/{product_id}
PUT /products

# Inventory
GET /inventory/{product_id}
POST /inventory/reserve

# Cart/Order
POST /cart/add
POST /checkout
GET /orders/{order_id}

# Payment
POST /payments/initiate
POST /payments/verify
```

### 🗄️ Data Models

```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY,
  name TEXT,
  email TEXT UNIQUE,
  password_hash TEXT
);

-- Products
CREATE TABLE products (
  id UUID PRIMARY KEY,
  name TEXT,
  description TEXT,
  price DECIMAL,
  stock INT
);

-- Orders
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  user_id UUID,
  product_id UUID,
  quantity INT,
  status TEXT,
  created_at TIMESTAMP
);
```

---

## 4️⃣ DETAILED DESIGN DEEP DIVES (10-12 minutes)

- Sync APIs for user, product, cart, order
- Async flows via Kafka for:
  - Inventory updates
  - Notifications (order confirmed, shipped)
  - Payment confirmed → trigger fulfillment

---

## 5️⃣ SCALING & PERFORMANCE (3-4 minutes)

| Challenge            | Solution                             |
| -------------------- | ------------------------------------ |
| High QPS             | Load balancer, API Gateway           |
| Inventory contention | Redis caching + eventual consistency |
| Payment latency      | Async flow, retries, verification    |
| Data durability      | WAL + DB replication                 |

---

## 6️⃣ MONITORING & RELIABILITY (2-3 minutes)

- **Metrics**: Orders/min, Payment errors, DB usage
- **Tools**: Prometheus + Grafana
- **Strategies**:
  - Circuit Breakers for external APIs
  - Redundant DB nodes (read replicas)
  - Graceful degradation (cached catalog)

---

## 7️⃣ INTERVIEW TIPS & TRADE-OFFS

### Key Points

- Discuss trade-offs (e.g., strong vs. eventual consistency)
- Think failure scenarios
- Add observability from day one
- Talk about data partitioning & caching

### Follow-up Ideas

- Add recommendation system
- Integrate fraud detection
- Add multi-region deployments