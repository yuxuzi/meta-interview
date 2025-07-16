# News Feed System Design - 30-Minute Interview Preparation

## Interview Timeline (30 minutes)

### 1. Requirements Gathering (5 minutes)

#### Key Questions to Ask the Interviewer:
- "What's the scale? How many concurrent users during peak?"
- "What's more important - consistency or availability?"
- "What's the read/write ratio?"
- "Do we need real-time updates or eventual consistency is fine?"
- "What types of media should we support?"

#### Functional Requirements to Clarify:
- **Core Features**: Create/view posts, follow/unfollow users, like/retweet/reply
- **Timeline**: View personalized feed from followed users
- **Media**: Support for images, videos, links
- **Interactions**: Real-time engagement (likes, replies, retweets)
- **User Management**: Profile management, authentication

#### Non-Functional Requirements:
- **Scale**: 200M DAU, 1B posts/day, 2B feed requests/day
- **Read/Write Ratio**: 100:1 (read-heavy system)
- **Latency**: Timeline loads <100ms, new posts appear in real-time
- **Availability**: 99.9% uptime
- **Consistency**: Posts shouldn't be missing or duplicated

---

### 2. Capacity Estimation (5 minutes)

#### Traffic Estimates:
```
Daily Active Users: 200M
Posts per day: 1B (avg 5 posts/user)
Timeline requests: 2B/day
Media uploads: 300M/day

QPS Calculations:
- Write QPS: ~12K QPS (1B posts/day)
- Read QPS: 1.2M QPS (100B timeline requests/day)
- Peak traffic: 3x average = 3.6M read QPS, 36K write QPS
```

#### Storage Estimates:
```
Post Data:
- Text: 280 chars = ~500 bytes
- Metadata: ~1KB per post
- Total per post: ~1.5KB
- Daily: 1B posts × 1.5KB = 1.5TB/day

Media Storage:
- 300M uploads/day × 1MB avg = 300TB/day
- Need CDN + object storage (S3)

Cache Requirements:
- Hot timeline data: 20% of users = 40M × 10KB = 400GB
- Media cache: 80% hit rate for popular content
```

---

### 3. High-Level Architecture (10 minutes)

#### System Components:

```
Client Apps (Web, Mobile)
        ↓
API Gateway (Auth, Rate Limiting, Load Balancing)
        ↓
┌─────────────┬─────────────┬─────────────┬─────────────┐
│ User Service│Tweet Service│Timeline Svc │Media Service│
│             │             │             │             │
└─────────────┴─────────────┴─────────────┴─────────────┘
        ↓            ↓            ↓            ↓
┌─────────────┬─────────────┬─────────────┬─────────────┐
│ User DB     │ Tweet DB    │Timeline DB  │ Object Store│
│ (MySQL)     │ (Cassandra) │ (Redis)     │ (S3 + CDN)  │
└─────────────┴─────────────┴─────────────┴─────────────┘
        ↓            ↓            ↓            ↓
    Message Queue (Kafka) → Fanout Workers → Notification Service
```

#### Core Services:

1. **User Service**: Authentication, profile management, follow/unfollow
2. **Tweet Service**: Create, store, and retrieve tweets
3. **Timeline Service**: Generate personalized feeds
4. **Media Service**: Handle image/video uploads and delivery
5. **Engagement Service**: Handle likes, retweets, replies
6. **Notification Service**: Push notifications for interactions
7. **Fanout Service**: Distribute new posts to followers' timelines

#### Database Design:

```sql
-- Users Table
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(100),
    created_at TIMESTAMP,
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0
);

-- Tweets Table
CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    content TEXT,
    media_urls JSON,
    created_at TIMESTAMP,
    like_count INT DEFAULT 0,
    retweet_count INT DEFAULT 0,
    reply_count INT DEFAULT 0,
    INDEX idx_user_time (user_id, created_at)
);

-- Follows Table
CREATE TABLE follows (
    follower_id BIGINT,
    following_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (follower_id, following_id)
);

-- Timeline Cache (Redis)
Key: timeline:user_id
Value: [tweet_id1, tweet_id2, ...] (sorted by timestamp)
```

---

### 4. Deep Dive: Timeline Generation Strategy (8 minutes)

#### The Fan-out Problem:
**Challenge**: When a user posts a tweet, how do we update all followers' timelines?

#### Solution: Hybrid Approach

**1. Fan-out on Write (Push Model)**
- **Use for**: Regular users (<1M followers)
- **Process**: When user posts → immediately write to all followers' timeline caches
- **Pros**: Fast read (timeline already computed)
- **Cons**: Slow write for popular users, storage intensive

**2. Fan-out on Read (Pull Model)**
- **Use for**: Celebrity users (>1M followers)
- **Process**: When user requests timeline → fetch from followed users in real-time
- **Pros**: Fast write, less storage
- **Cons**: Slower read, need to aggregate multiple sources

**3. Hybrid Implementation**:
```python
def post_tweet(user_id, content):
    # Store tweet
    tweet_id = tweet_service.create_tweet(user_id, content)
    
    # Get followers
    followers = user_service.get_followers(user_id)
    
    if len(followers) < CELEBRITY_THRESHOLD:
        # Fan-out on write for regular users
        fanout_service.push_to_followers(tweet_id, followers)
    else:
        # Store in celebrity feed for pull-based retrieval
        celebrity_feed.add_tweet(user_id, tweet_id)

def get_timeline(user_id):
    timeline = []
    
    # Get from precomputed timeline (push)
    timeline.extend(timeline_cache.get(user_id))
    
    # Get from celebrity users (pull)
    celebrities = user_service.get_followed_celebrities(user_id)
    for celebrity in celebrities:
        recent_tweets = celebrity_feed.get_recent(celebrity.id)
        timeline.extend(recent_tweets)
    
    # Merge and sort by timestamp
    return sort_by_timestamp(timeline)
```

#### Timeline Ranking Algorithm:
- **Chronological**: Default sort by timestamp
- **Engagement-based**: Factor in likes, retweets, replies
- **Personalization**: ML model considering user behavior, relationships
- **Freshness**: Boost recent content

---

### 5. Scalability & Optimization (2 minutes)

#### Caching Strategy:
- **Timeline Cache**: Redis with 1-hour TTL for active users
- **Tweet Cache**: Cache popular tweets with 24-hour TTL
- **Media CDN**: Global CDN for images/videos
- **Database Read Replicas**: Distribute read load

#### Performance Optimizations:
- **Pagination**: Load 20 tweets per request, lazy loading
- **Async Processing**: Use Kafka for fanout jobs, notifications
- **Database Sharding**: Shard by user_id for tweets and timelines
- **Load Balancing**: Consistent hashing for service distribution

#### Monitoring & Metrics:
- **Latency**: P95 timeline load time <100ms
- **Throughput**: Track QPS per service
- **Error Rates**: Monitor 4xx/5xx responses
- **Cache Hit Rates**: Maintain >80% hit rate

---

## Key APIs

```python
# Core APIs to discuss
POST /api/v1/tweets
GET /api/v1/timeline?user_id=123&limit=20&offset=0
POST /api/v1/users/follow
POST /api/v1/tweets/{tweet_id}/like
POST /api/v1/media/upload
GET /api/v1/tweets/{tweet_id}
```

## Common Interview Questions & Answers

**Q: How do you handle a celebrity with 100M followers posting?**
A: Use fan-out on read for celebrities. Store their tweets separately and fetch during timeline generation rather than pushing to all followers.

**Q: How do you ensure timeline consistency?**
A: Use eventual consistency with proper ordering. Cache invalidation and refresh mechanisms ensure users eventually see all relevant content.

**Q: How do you handle media uploads?**
A: Async upload to S3, generate multiple sizes/formats, use CDN for delivery, and store metadata in tweet database.

**Q: What if the timeline service goes down?**
A: Implement circuit breakers, fallback to direct database queries, use multiple regions with data replication.

## Final Tips for the Interview

1. **Start simple**: Begin with basic architecture, then add complexity
2. **Ask clarifying questions**: Don't make assumptions about requirements
3. **Discuss trade-offs**: Every design decision has pros and cons
4. **Consider edge cases**: Celebrity users, viral content, system failures
5. **Think about monitoring**: How would you know if the system is working?
6. **Be prepared for follow-ups**: Interviewer may ask about specific components
7. **Practice drawing**: Be comfortable sketching architecture diagrams

.