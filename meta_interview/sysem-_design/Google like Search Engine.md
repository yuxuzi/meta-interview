# Google-like Search Engine Design — 30-Minute Interview Guide

## 1️⃣ REQUIREMENTS CLARIFICATION (3-4 minutes)

### 🎯 Core Objective
Design a web search engine that can index billions of web pages and serve search queries with sub-second latency.

### ✅ Functional Requirements
- **Web Crawling**: Discover and fetch web pages across the internet
- **Indexing**: Process and store searchable content efficiently  
- **Query Processing**: Parse user queries and return relevant results
- **Ranking**: Order results by relevance and authority
- **Real-time Updates**: Handle new/updated content continuously

### 🔒 Non-Functional Requirements
- **Scale**: 100M daily active users, 10B pages indexed
- **Performance**: <200ms query response time
- **Availability**: 99.99% uptime
- **Throughput**: 100K queries per second (peak)
- **Storage Efficiency**: Petabyte-scale data management

---

## 2️⃣ CAPACITY ESTIMATION (4-5 minutes)

### 📊 Traffic & Storage Analysis

| Metric | Estimate | Calculation |
|--------|----------|-------------|
| **Daily Active Users** | 100M | Given |
| **Queries per User/Day** | 50 | Industry average |
| **Total Daily Queries** | 5B | 100M × 50 |
| **Peak QPS** | 100K | 5B ÷ 86,400 × 2 (peak factor) |
| **Average QPS** | 50K | 5B ÷ 86,400 |

### 💾 Storage Requirements

| Component | Size | Notes |
|-----------|------|-------|
| **Raw Web Pages** | 10 PB | 10B pages × 1MB average |
| **Processed Content** | 3-5 PB | After cleaning/tokenization |
| **Inverted Index** | 500-800 GB | Compressed tokens → document mapping |
| **Forward Index** | 5 PB | Document metadata and content |
| **Link Graph** | 2 PB | Page relationships for PageRank |
| **Cache Layer** | 100-500 GB | Hot query results |

### 🕷️ Crawling Capacity
- **Target**: 10M pages/day for fresh content
- **Crawl Rate**: ~115 pages/second
- **Crawler Fleet**: 1000 distributed crawlers
- **Per Crawler**: 0.1-0.2 pages/second (respecting robots.txt)

---

## 3️⃣ HIGH-LEVEL ARCHITECTURE (8-10 minutes)

### 🏗️ System Architecture Diagram

```
                           GOOGLE-LIKE SEARCH ENGINE ARCHITECTURE
                                           
┌─────────────────────────────────────────────────────────────────────────────┐
│                           👥 USER INTERFACE LAYER                          │
├─────────────────┬─────────────────┬─────────────────┬─────────────────────┤
│  Web Frontend   │   Mobile Apps   │   API Gateway   │    Load Balancer    │
│  (React/Vue)    │  (iOS/Android)  │ (Rate Limiting) │   (Nginx/HAProxy)   │
└─────────────────┴─────────────────┴─────────────────┴─────────────────────┘
                                    ↓ 100K QPS
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ⚡ QUERY PROCESSING LAYER                            │
├─────────────────┬─────────────────┬─────────────────┬─────────────────────┤
│  Query Parser   │  Query Router   │   Cache Layer   │  Result Formatter   │
│ (NLP, Speller)  │ (Shard Router)  │ (Redis Cluster) │ (Snippet Generator) │
└─────────────────┴─────────────────┴─────────────────┴─────────────────────┘
                                    ↓ Index Lookup
┌─────────────────────────────────────────────────────────────────────────────┐
│                         📚 INDEX & SEARCH LAYER                            │
├─────────────────┬─────────────────┬─────────────────┬─────────────────────┤
│ Inverted Index  │ Forward Index   │ Ranking Engine  │   Link Graph DB     │
│ (Elasticsearch) │ (Document Meta) │(PageRank+ML)    │  (Graph Database)   │
│   500-800 GB    │      5 PB       │                 │       2 PB          │
└─────────────────┴─────────────────┴─────────────────┴─────────────────────┘
                                    ↓ Content Feed
┌─────────────────────────────────────────────────────────────────────────────┐
│                       🕷️ CRAWLING & INDEXING LAYER                         │
├─────────────────┬─────────────────┬─────────────────┬─────────────────────┤
│   URL Manager   │  Crawler Fleet  │Content Processor│   Index Builder     │
│(Queue+Schedule) │(1000 Workers)   │(Text Extraction)│  (MapReduce)        │
│                 │ 115 pages/sec   │                 │                     │
└─────────────────┴─────────────────┴─────────────────┴─────────────────────┘
                                    ↓ Data Storage
┌─────────────────────────────────────────────────────────────────────────────┐
│                           💾 STORAGE LAYER                                 │
├─────────────────┬─────────────────┬─────────────────┬─────────────────────┤
│ Document Store  │  Index Storage  │  Metadata DB    │   Backup & Replica  │
│   HDFS/GFS      │   NoSQL Shards  │  PostgreSQL     │   Cross-DC Sync     │
│     10 PB       │                 │                 │                     │
└─────────────────┴─────────────────┴─────────────────┴─────────────────────┘
```

### 🏗️ Core Components

| Component | Responsibility | Technology Choice |
|-----------|---------------|-------------------|
| **URL Manager** | Crawl queue, deduplication, scheduling | Redis + PostgreSQL |
| **Crawler Service** | Fetch pages, extract content | Distributed Go/Java workers |
| **Content Processor** | Clean HTML, extract text/links | Apache Kafka + Spark |
| **Indexer** | Build inverted/forward indexes | Elasticsearch/Lucene |
| **Query Service** | Parse queries, retrieve results | Custom service + cache |
| **Ranking Engine** | Score and rank results | ML models + PageRank |
| **Storage Layer** | Distributed file system | HDFS/GFS + NoSQL |

### 🌐 API Design

```http
# Search API
GET /search?q={query}&page={n}&filters={type}
Response: {
  "results": [...],
  "total": 1000000,
  "time_ms": 150
}

# Crawler Management
POST /crawler/submit
Body: {"urls": ["http://example.com"]}

# Index Stats
GET /index/stats
Response: {"documents": 10000000000, "size_gb": 800000}
```

### 🗄️ Data Models

#### URL Queue Schema
```sql
CREATE TABLE url_queue (
  url_hash VARCHAR(64) PRIMARY KEY,
  url TEXT NOT NULL,
  domain VARCHAR(255),
  next_crawl_time TIMESTAMP,
  priority INT DEFAULT 5,
  retry_count INT DEFAULT 0,
  status ENUM('pending', 'crawling', 'done', 'failed')
);
```

#### Document Index Schema
```json
{
  "doc_id": "sha256_hash",
  "url": "https://example.com/page",
  "title": "Page Title",
  "content_tokens": ["word1", "word2"],
  "outbound_links": ["url1", "url2"],
  "pagerank_score": 0.85,
  "last_crawled": "2025-07-25T10:00:00Z",
  "language": "en"
}
```

#### Inverted Index Structure
```
Token: "machine learning"
Posting List: [
  {doc_id: "abc123", tf: 5, positions: [10, 45, 100]},
  {doc_id: "def456", tf: 3, positions: [5, 200, 350]},
  ...
]
```

---

## 4️⃣ DETAILED DESIGN DEEP DIVES (10-12 minutes)

### 🕷️ Web Crawling System

**Architecture Pattern**: Producer-Consumer with Priority Queues

**Crawl Strategy**:
1. **Seed URLs** → URL Manager assigns to crawler workers
2. **Politeness**: Respect robots.txt, rate limiting per domain
3. **Deduplication**: Content hashing to avoid duplicate processing
4. **Freshness**: Priority-based recrawling (news sites hourly, static sites monthly)

**Key Optimizations**:
- **Domain Partitioning**: Distribute crawlers by domain hash
- **Bloom Filters**: Fast duplicate URL detection
- **Content Diffing**: Only reindex changed content

### 📚 Indexing Pipeline

**Processing Flow**:
```
Raw HTML → Content Extraction → Tokenization → Index Building → Storage
```

**Inverted Index Design**:
- **Sharding Strategy**: Partition by term hash for load distribution
- **Compression**: Delta encoding for document IDs, term frequency
- **Update Strategy**: Incremental updates + periodic full rebuilds

**Forward Index Benefits**:
- Fast document retrieval for result snippets
- Metadata storage (title, URL, timestamp)
- Link analysis data for PageRank

### 🔍 Query Processing Engine

**Query Flow**:
1. **Parse**: Extract terms, operators, filters
2. **Optimize**: Query rewriting, spell correction
3. **Retrieve**: Parallel lookup across index shards
4. **Intersect**: Combine posting lists efficiently
5. **Rank**: Apply relevance scoring algorithms
6. **Format**: Generate result snippets and metadata

**Ranking Factors**:
- **TF-IDF**: Term frequency × inverse document frequency
- **PageRank**: Link-based authority scoring
- **Freshness**: Recent content boost
- **User Signals**: Click-through rates, dwell time
- **Query Context**: Location, personalization

### ⚡ Caching Strategy

**Multi-Layer Caching**:
- **L1 - Query Cache**: Redis cluster for popular queries (TTL: 5-15 minutes)
- **L2 - Index Cache**: Hot index segments in memory
- **L3 - CDN**: Geographically distributed static content

**Cache Invalidation**:
- **Time-based**: TTL for different content types
- **Event-driven**: Real-time updates for breaking news
- **LRU Eviction**: Memory management for hot data

---

## 5️⃣ SCALING & PERFORMANCE (3-4 minutes)

### 🎯 Performance Optimizations

| Challenge | Solution | Impact |
|-----------|----------|---------|
| **Query Latency** | Index sharding + parallel processing | <200ms response |
| **Storage Costs** | Compression + tiered storage | 60% reduction |
| **Crawl Efficiency** | Distributed crawlers + smart scheduling | 10M pages/day |
| **Index Updates** | Incremental indexing + background merging | Real-time freshness |

### 🔧 Bottleneck Resolutions

**Crawling Bottlenecks**:
- Bandwidth limits → Geographically distributed crawlers
- Site rate limits → Adaptive backoff strategies
- Duplicate detection → Bloom filters + content hashing

**Indexing Bottlenecks**:
- Memory usage → Segment-based indexing with compression
- I/O throughput → SSD storage + batch processing
- Update conflicts → Write-ahead logging + eventual consistency

**Query Bottlenecks**:
- High QPS → Horizontal sharding + load balancing
- Complex queries → Query optimization + result caching
- Global distribution → Regional data centers + CDN

---

## 6️⃣ MONITORING & RELIABILITY (2-3 minutes)

### 📊 Key Metrics
- **Crawl Health**: Pages/second, error rates, coverage
- **Index Freshness**: Average document age, update lag
- **Query Performance**: Latency P95/P99, QPS, cache hit rate
- **System Health**: CPU/memory usage, disk I/O, network throughput

### 🛡️ Fault Tolerance
- **Data Replication**: 3x replication across availability zones
- **Service Redundancy**: Load balancers + auto-scaling groups
- **Graceful Degradation**: Serve stale results during outages
- **Circuit Breakers**: Prevent cascade failures

---

## 7️⃣ INTERVIEW TIPS & TRADE-OFFS

### ✅ Key Discussion Points
1. **Consistency vs. Availability**: Eventual consistency for better performance
2. **Cost vs. Freshness**: Balance crawl frequency with infrastructure costs
3. **Relevance vs. Speed**: Complex ranking algorithms vs. query latency
4. **Centralized vs. Distributed**: Trade coordination complexity for scalability

### 🎯 Impressive Details to Mention
- **MapReduce** for distributed index building
- **Consistent hashing** for shard distribution
- **Bloom filters** for memory-efficient duplicate detection
- **Machine learning** for query understanding and ranking
- **A/B testing** framework for ranking algorithm improvements

### 🚀 Follow-up Extensions
- **Image/Video search** capabilities
- **Real-time search** for social media content  
- **Personalization** and user behavior modeling
- **Mobile optimization** and voice search
- **Knowledge graph** integration for entity search

This design handles 100M users with sub-200ms latency while maintaining 99.99% availability through distributed architecture, intelligent caching, and fault-tolerant design patterns.