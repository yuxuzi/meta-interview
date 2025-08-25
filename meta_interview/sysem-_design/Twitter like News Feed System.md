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



.# 🎯 System Design Interview: News Feed System
## Realistic Interview Conversation

---

## Phase 1: Requirements Gathering (5 minutes)

**Interviewer:** "Alright, so today we're going to design a news feed system - think Twitter or Facebook's main feed. Before we jump in, what do you think are the core things this system needs to do?"

**Candidate:** "Right, so when I think about a news feed system, the main things that come to mind are - users need to be able to post content, they need to follow other users, and then they need to see a personalized feed of posts from people they follow. Also, people want to interact with posts - like them, share them, comment on them."

**Interviewer:** "Exactly! So what about non-functional requirements? What are some constraints we should think about?"

**Candidate:** "Well, this needs to handle a lot of users and a lot of traffic. The system needs to be highly available - if Facebook goes down, that's a big problem. Also, when someone posts something, their followers should see it pretty quickly, though it doesn't need to be instant. And when I open my feed, I expect it to load fast - definitely under a second, ideally much faster."

**Interviewer:** "Good thinking. Now let me give you some numbers to work with. We're looking at about 200 million daily active users, roughly 1 billion posts per day, and people read way more than they write - let's say 100 times more reading than writing. What questions does this raise for you?"

**Candidate:** "Wow, okay, so we're talking massive scale here. A few things I'm wondering - are we supporting just text posts or also images and videos? Because that changes storage requirements dramatically. Also, when you say the feed should load quickly, what's our target? And how personalized does this feed need to be - is it just chronological, or do we need some kind of ranking algorithm?"

**Interviewer:** "Let's say we need to support images and videos, feed load time should be under 100 milliseconds, and yes, we want some kind of intelligent ranking, not just chronological order."

**Candidate:** "Got it. So we're building a system that's heavily read-optimized, needs to handle rich media, and has to be smart about what content to show users first. The scale means we definitely need to think about caching and probably can't generate feeds in real-time for every request."

---

## Phase 2: Capacity Estimation (5 minutes)

**Interviewer:** "Let's talk numbers. Can you walk me through some rough calculations for what we're dealing with here?"

**Candidate:** "Sure, let me think through this. So we have 200 million daily active users and 1 billion posts per day. That works out to about 12,000 new posts every second on average, but obviously it's going to spike during peak hours - let's say 3 times higher, so maybe 36,000 posts per second at peak."

**Interviewer:** "And what about the read side?"

**Candidate:** "Right, so if the read-to-write ratio is 100 to 1, we're looking at about 1.2 million read requests per second on average, and maybe 3.6 million at peak. That's a lot of timeline requests to handle."

**Interviewer:** "What about storage?"

**Candidate:** "Hmm, let me think. If each post is maybe 1-2 KB of text data and metadata, that's about 1.5 terabytes of text data per day. But the real killer is going to be media. If even 30% of posts have images or videos, and let's say those average 100 KB each, that's 30 terabytes per day just for media. Over a few years, we're talking petabytes of storage."

**Interviewer:** "Those are big numbers. How does that influence your architecture thinking?"

**Candidate:** "Well, it definitely means we can't put everything in one database. We'll need different storage solutions - probably something like a distributed NoSQL database for the post data since we have so many writes, and definitely object storage with a CDN for all the media files. And for the timeline generation, we'll need heavy caching because we can't afford to compute 3.6 million personalized feeds from scratch every second."

---

## Phase 3: High-Level Architecture (10 minutes)

**Interviewer:** "Okay, let's start designing this thing. What are the main components you'd need?"

**Candidate:** "Alright, so I'm thinking we need several key services here. First, a user service that handles authentication, user profiles, and the follow relationships between users. Then we need a post service that handles creating and storing posts, including uploading media. The tricky part is the timeline service - that's what generates the personalized feed for each user."

**Interviewer:** "What else?"

**Candidate:** "We'll also need what I'd call a fanout service. When someone posts something, we need to figure out which of their followers should see it and get it into their feeds somehow. And probably a notification service for real-time updates. Oh, and we'll definitely need a media service to handle image and video processing, resizing, that sort of thing."

**Interviewer:** "How do these services talk to each other?"

**Candidate:** "I'd put an API gateway in front of everything to handle authentication and routing. The services would communicate through both synchronous API calls for things that need immediate responses, and asynchronous message queues for things like the fanout process - when someone posts, we don't want to make them wait while we push that post to all their followers' feeds."

**Interviewer:** "What about databases?"

**Candidate:** "I'm thinking we need a few different types. For user data and follow relationships, a traditional SQL database like MySQL or PostgreSQL makes sense because we need strong consistency for things like 'who follows whom.' For the posts themselves, something like Cassandra would be better because it's designed for high write volume. And for the timeline caching, Redis would be perfect because it's super fast for reads."

**Interviewer:** "Tell me about your API design. What would a few key endpoints look like?"

**Candidate:** "Sure, so we'd have something like POST /posts to create a new post, GET /timeline for getting a user's personalized feed, POST /users/123/follow for following someone, and POST /posts/456/like for liking a post. For the timeline endpoint, we'd want pagination, so maybe GET /timeline?limit=20&cursor=abc123 to get the next batch of posts."

---

## Phase 4: Deep Dive - Timeline Generation (8 minutes)

**Interviewer:** "The timeline generation is really the heart of this system. Walk me through how you'd actually build someone's feed when they open the app."

**Candidate:** "This is definitely the most interesting part! So there are basically two approaches you can take. The first is what I'd call 'push' - every time someone posts, you immediately push that post into the pre-computed timelines of all their followers. The second is 'pull' - when someone opens their app, you go find all the people they follow, grab their recent posts, and build the timeline on the spot."

**Interviewer:** "What are the tradeoffs?"

**Candidate:** "Well, push is great for read performance because the timeline is already built and sitting in cache, but it's terrible when someone has millions of followers because you'd have to update millions of timelines every time they post. Pull is the opposite - fast to post, but slow to read because you're doing all that work when the user is waiting."

**Interviewer:** "So which would you choose?"

**Candidate:** "I'd actually do a hybrid approach. For regular users with a reasonable number of followers - let's say under a million - I'd use the push model. When they post, we'd asynchronously push that post into their followers' cached timelines. But for celebrities with millions of followers, I'd use the pull model."

**Interviewer:** "How would that work in practice?"

**Candidate:** "So when a user opens their feed, we'd first check their cached timeline, which has posts from all the regular people they follow. Then we'd also check if they follow any celebrities, and if so, we'd pull the latest posts from those celebrities and merge them in. This way, we get the best of both worlds."

**Interviewer:** "But now you have to merge and rank posts from different sources. How do you decide what order to show them in?"

**Candidate:** "Right, that's the ranking algorithm. I'd consider several factors - recency is important, so newer posts get a boost. Engagement matters too, so posts with more likes or comments get ranked higher. I'd also look at the user's past behavior - if they always like posts from a particular friend, show more of that friend's content. And maybe factor in content type - if someone always watches videos, prioritize video posts for them."

**Interviewer:** "What happens if your caching layer fails?"

**Candidate:** "That's a great point. If Redis goes down, we can't just show users empty feeds. I'd implement a fallback where if we can't get the cached timeline, we fall back to the pull model temporarily. It'll be slower, but at least users still get their content. We'd also want Redis to be clustered with replication so it's less likely to fail completely."

**Interviewer:** "How do you handle the consistency issues? What if someone unfollows someone else but still sees their posts for a while?"

**Candidate:** "Yeah, that's the tradeoff with caching. I think for a social media system, eventual consistency is acceptable. If you unfollow someone and still see their posts for a few minutes, that's not the end of the world. We could set cache TTLs to maybe an hour, so things would correct themselves fairly quickly. For more immediate consistency, we could invalidate cache entries when follow relationships change, but that adds complexity."

---

## Phase 5: Scaling & Final Considerations (2 minutes)

**Interviewer:** "We're almost out of time. What do you see as the main scaling bottlenecks, and how would you handle them?"

**Candidate:** "The biggest bottleneck is probably the timeline generation itself. Even with caching, serving 3.6 million timeline requests per second is no joke. I'd want to have multiple Redis clusters, probably sharded by user ID, and multiple timeline service instances behind a load balancer."

**Interviewer:** "What about the write side?"

**Candidate:** "The fanout process could also become a bottleneck. When a celebrity posts, we might need to update millions of cache entries. I'd use a message queue system like Kafka to make this asynchronous, and have multiple worker processes consuming from the queue. We could also batch updates to be more efficient."

**Interviewer:** "Any other considerations?"

**Candidate:** "Media is going to be huge - both in terms of storage and bandwidth. We'd definitely want a CDN with edge locations worldwide so users get images and videos from servers close to them. And we'd want different video qualities for different devices and connection speeds."

**Interviewer:** "What about monitoring?"

**Candidate:** "Oh absolutely. We'd want to monitor timeline generation latency, cache hit rates, the length of our message queues for fanout processing, and definitely track any spikes in error rates. If timeline generation starts taking longer than 100ms, we need to know immediately."

**Interviewer:** "Great! Any final thoughts?"

**Candidate:** "I think the key insight here is that this is fundamentally a caching problem at scale. The raw data processing isn't that complex, but pre-computing and caching the right things so that 200 million people can get their feeds in under 100ms - that's the real challenge. Everything else - the databases, the APIs, the media handling - that's all pretty standard stuff, just at a large scale."

**Interviewer:** "Perfect. That's a solid design. Thanks for walking through that with me!"

---

## 💡 Key Interview Tips Demonstrated:

- **Started with clarifying questions** before diving into solutions
- **Broke down the problem systematically** (functional → non-functional → scale)
- **Did back-of-envelope math** to understand the scope
- **Acknowledged tradeoffs** (consistency vs. performance, push vs. pull)
- **Thought about failure scenarios** and fallback strategies
- **Considered monitoring and operational concerns**
- **Used conversational language** rather than overly technical jargon
- **Built up the solution incrementally** rather than presenting a complete design upfront
- **Asked follow-up questions** to show engagement and thoroughness