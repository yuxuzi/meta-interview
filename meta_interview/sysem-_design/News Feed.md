# 📰 News Feed System Design – Standout 30-Minute Interview Solution

## ⏱ Interview Agenda

| Phase | Duration | Focus |
|-------|----------|-------|
| 1. Requirements Gathering | 5 min | Core features + clarifying Qs |
| 2. Capacity Estimation     | 5 min | Scale, QPS, storage |
| 3. High-Level Architecture | 10 min| Component overview |
| 4. Deep Dive               | 8 min | Timeline generation |
| 5. Final Notes & Scaling   | 2 min | Performance & wrap-up |

---

## 1. 🎯 Requirements Gathering

### ✅ Functional Requirements
- Create/view posts
- Follow/unfollow users
- Like/retweet/reply interactions
- View personalized feed
- Support media: images, videos, links
- Real-time engagement updates

### 🔒 Non-Functional Requirements
- Scale: 200M DAU, 1B posts/day, 2B feed requests/day
- Read/Write Ratio: 100:1 (heavily read-optimized)
- Latency: <100ms timeline load, real-time post appearance
- Availability: 99.9%
- Consistency: No missing/duplicate posts

### ❓ Clarifying Questions
- What’s peak concurrent user count?
- Preference: consistency vs. availability?
- Is real-time feed required, or is eventual consistency okay?
- What media types should we support?
- Timeline ranking: chronological vs. personalized?

---

## 2. 📈 Capacity Estimation

### Traffic & QPS

| Metric | Estimate |
|--------|----------|
| DAU | 200M |
| Posts/day | 1B |
| Timeline requests/day | 2B |
| Media uploads/day | 300M |
| Avg Write QPS | ~12K |
| Avg Read QPS | ~1.2M |
| Peak QPS | 3x Avg → 3.6M reads, 36K writes |

### Storage Estimates
- Post: ~1.5KB/post × 1B → **1.5TB/day**
- Media: ~300TB/day → **Object store + CDN**
- Timeline Cache: 40M users × 10KB → **~400GB**

---

## 3. 🏗 High-Level Architecture

```
Client Apps (Web/Mobile)
      ↓
API Gateway (Auth, Rate Limit, Routing)
      ↓
 ┌─────────────┬─────────────┬─────────────┬─────────────┐
 │ User Svc    │ Tweet Svc   │ Timeline Svc│ Media Svc   │
 └─────────────┴─────────────┴─────────────┴─────────────┘
      ↓             ↓             ↓             ↓
 ┌─────────────┬─────────────┬─────────────┬─────────────┐
 │ User DB     │ Tweet DB    │ Timeline DB │ Object Store│
 │ (MySQL)     │ (Cassandra) │ (Redis)     │ (S3 + CDN)  │
 └─────────────┴─────────────┴─────────────┴─────────────┘
                       ↓
              Kafka → Fanout Workers → Notification Svc
```

### Core Services
- **User Service**: Auth, profile, follow graph
- **Tweet Service**: Post CRUD, engagement counts
- **Timeline Service**: Personal feed generation
- **Media Service**: Upload, transform, deliver
- **Fanout/Notification**: Distribute new posts to followers
- **Engagement Service**: Likes, retweets, replies

---

## 4. 🔬 Deep Dive — Timeline Generation

### 🧩 Hybrid Fanout Strategy

#### 1. Fan-out on Write (Push)
- 👥 Used for regular users (<1M followers)
- Posts pushed to Redis cache of all followers
- ✅ Fast reads, high cache hit

#### 2. Fan-out on Read (Pull)
- 🌟 Used for celebrities (>1M followers)
- Followers pull tweets at read time from tweet DB
- ✅ Fast writes, scalable for viral content

#### 3. Combined Model
```python
def post_tweet(user_id, content):
    tweet_id = tweet_service.save(user_id, content)
    followers = user_service.get_followers(user_id)
    
    if len(followers) < CELEBRITY_THRESHOLD:
        fanout.push(tweet_id, followers)
    else:
        celebrity_feed.store(user_id, tweet_id)

def get_timeline(user_id):
    push_feed = redis.get(f"user:{user_id}:timeline")
    celeb_ids = user_service.get_celebrity_follows(user_id)
    
    pull_feed = []
    for celeb_id in celeb_ids:
        pull_feed += celebrity_feed.get_recent(celeb_id)
    
    timeline = sort(push_feed + pull_feed)
    return timeline
```

### 🧠 Feed Ranking Strategies
- **Chronological**
- **Engagement-based**: Likes, replies, retweets
- **Personalized**: User behavior, relevance scores
- **Freshness Boost**: Prioritize new content

---

## 5. ⚙️ Scaling, Performance & Reliability

### 🔥 Bottlenecks & Mitigations

| Area | Challenge | Solution |
|------|-----------|----------|
| Fanout | Slow writes | Kafka + Worker pool |
| Tweet DB | Write hotspots | Cassandra + Sharding |
| Timeline Cache | Large cache size | Redis + Eviction TTL |
| Media Delivery | CDN scaling | S3 + CloudFront |

### 🗃️ Caching
- Timeline: Redis with 1h TTL
- Tweets: Hot content with 24h TTL
- Media: CDN edge caching

### 🚀 Optimizations
- Pagination & lazy loading
- Kafka async fanout/engagements
- Consistent hashing for load balancing
- Read replicas for MySQL

### 📈 Monitoring
- P95 latency <100ms
- Cache hit rate >80%
- QPS dashboard per service
- 4xx/5xx error tracking

---

## 🔌 Key APIs

```python
POST /api/v1/tweets
GET /api/v1/timeline?user_id=123&limit=20
POST /api/v1/users/follow
POST /api/v1/tweets/{id}/like
POST /api/v1/media/upload
GET /api/v1/tweets/{id}
```

---

## 🧠 Smart Interview Highlights

- "For celebrities, I use fan-out on read to avoid mass writes."
- "Timeline consistency is eventually guaranteed via deduplicated fetch and TTL cache."
- "Media uploads handled asynchronously with virus scanning & CDN resize pipeline."
- "Each component is horizontally scalable and stateless wherever possible."

---

## ❓ Interview Follow-ups to Anticipate

- How do you rank feeds for engagement?
- What if Redis fails—fallback plan?
- How do you handle real-time likes?
- How would this differ for enterprise use (LinkedIn-style)?
- What changes for threaded posts or comments?



.