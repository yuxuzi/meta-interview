# 🎬 YouTube-like System Design — 30-Min Interview Guide

## 1️⃣ REQUIREMENTS

### 🎯 Objective
Design a simplified video sharing platform that supports:
- User registration and login
- Video upload, storage, and streaming
- Video search by metadata (title, tags)
- Likes, dislikes, and comments

### ✅ Functional Requirements
- User accounts (register/login)
- Upload & encode video files
- Stream videos in multiple resolutions
- Search videos by title/tags
- Like, dislike, and comment

### 🔒 Non-Functional Requirements
- Scalability: 100M daily active users
- Low latency: <1s stream & search
- High availability: 99.99% uptime
- Durability: No video loss
- Cost-efficiency: Optimize storage & delivery

---

## 2️⃣ CAPACITY ESTIMATION
Assumptions and Estimations





Daily Active Users: 100 million.



Video Size: Average 500MB per video.



Upload Rate: ~500 hours of video per minute (YouTube’s real-world scale, scaled down for simplicity).



Read:Write Ratio: 100:1 (100 reads per write, e.g., views vs. uploads).



Query Per Second (QPS): ~10 billion read requests per day (100M users × 100 reads ÷ 86,400 seconds).



Storage Needs: If 500 hours of video (at 500MB per 5-minute video) are uploaded per minute, storage grows rapidly (~6PB per day).



| Metric                 | Estimate                         |
|------------------------|----------------------------------|
| DAU                    | 100M                             |
| Upload rate            | 500 hrs/min (~6PB/day)           |
| Avg video size         | 500MB                            |
| Read:Write ratio       | 100:1                            |
| Read QPS               | ~115K (10B/day)                  |
| Video formats          | 480p, 720p, 1080p                |

---

## 3️⃣ HIGH-LEVEL DESIGN

### 📌 Components

| Component           | Responsibility                                  |
|---------------------|--------------------------------------------------|
| User Service        | User registration, login, auth                   |
| Video Service       | Upload, metadata, status                         |
| Transcoding Worker  | Encode to 480p/720p/1080p                        |
| Storage Service     | Store raw + encoded video blobs (e.g., S3)       |
| Search Service      | Search by title, description, tags               |
| Interaction Service | Handle likes/dislikes/comments                   |
| CDN                 | Serve video content globally                     |
| Redis               | Cache metadata, search queries, interaction counts|
| Kafka               | Queue encoding & background jobs                 |

---

### 🧭 Flow Diagram (Text)
```
[User] <--> [Web Server/Mobile App]
          |
          v
[Load Balancer] <--> [API Server] <--> [Database] (e.g., MySQL/Cassandra)
          |                  |
          |                  v
          |            [Caching Layer] (e.g., Redis)
          |
          v
[Video Upload] --> [Video Service] --> [Storage Service] (e.g., AWS S3)
          |                  |
          |                  v
          |            [Transcoding Service] --> [Storage Service]
          |
          v
[Video Streaming] --> [CDN] --> [User]
          |
          v
[Search] --> [Search Service] (e.g., Elasticsearch) --> [Database]
          |
          v
[Interactions] --> [Interaction Service] --> [Database]
```

---

### 🌐 API Design (REST)

| Endpoint                          | Method | Description                         |
|----------------------------------|--------|-------------------------------------|
| `/api/register`                  | POST   | Create new user                     |
| `/api/login`                     | POST   | Authenticate user                   |
| `/api/videos/upload`             | POST   | Upload video chunks                 |
| `/api/videos/{id}`               | GET    | Fetch metadata + stream URL         |
| `/api/videos/{id}/like`          | POST   | Like/dislike video                  |
| `/api/videos/{id}/comments`      | POST   | Post a comment                      |
| `/api/videos/{id}/comments`      | GET    | Get comments                        |
| `/api/search?q=keyword`          | GET    | Search videos by metadata           |

---

### 🧩 Database Design

#### 👤 User Table
| Field         | Type     |
|---------------|----------|
| user_id       | UUID     |
| username      | VARCHAR  |
| email         | VARCHAR  |
| password_hash | TEXT     |
| created_at    | TIMESTAMP|

#### 📼 Video Table
| Field         | Type     |
|---------------|----------|
| video_id      | UUID     |
| user_id       | UUID     |
| title         | VARCHAR  |
| description   | TEXT     |
| tags          | TEXT[]   |
| views         | INT      |
| status        | ENUM     |
| created_at    | TIMESTAMP|

#### 💬 Comment Table
| Field         | Type     |
|---------------|----------|
| comment_id    | UUID     |
| video_id      | UUID     |
| user_id       | UUID     |
| content       | TEXT     |
| created_at    | TIMESTAMP|

#### 👍 Like Table (Denormalized)
| Field         | Type     |
|---------------|----------|
| video_id      | UUID     |
| like_count    | INT      |
| dislike_count | INT      |

#### 🔍 Elasticsearch Index
| Field       | Example                           |
|-------------|------------------------------------|
| video_id    | uuid-123                           |
| title       | "System Design for YouTube"        |
| description | "Learn how to build YouTube"       |
| tags        | ["system-design", "youtube"]       |

---

## 4️⃣ DEEP DIVES

### 📼 Upload & Encode

- User uploads → Video metadata saved → Kafka queues job
- Transcoder processes to 480p/720p/1080p
- Store output to S3
- Metadata status: UPLOADING → READY

### 🔍 Search

- Elasticsearch indexes title/tags/description
- Query with autocomplete + relevance
- Redis caches top search queries

### 📡 Stream

- CDN serves HLS/DASH chunks
- Pre-signed URLs for secure access
- ABR (adaptive bitrate) streaming

### 💬 Interactions

- Redis counters for likes/views → batch sync
- Comments in DB
- Use debounce/write batching

---

## 5️⃣ FINAL DESIGN SUMMARY

### ✅ Summary

- Microservice architecture
- Kafka decouples heavy jobs (encoding, indexing)
- CDN + Redis ensure performance
- S3 + tiered storage for cost/durability
- Elasticsearch powers search
- Eventual consistency for interactions

### ⚖️ Trade-offs

| Decision             | Why                                |
|----------------------|-------------------------------------|
| Eventual consistency | Scalability for likes/comments      |
| Async encoding       | UI responsiveness                   |
| CDN + transcoding    | Reliable, fast playback             |
| Redis + cache layers | <1s latency for hot content         |

### 🚀 Optional Additions

- Subscriptions, notifications
- ML recommendations
- Spam filtering / moderation
- Video monetization

---