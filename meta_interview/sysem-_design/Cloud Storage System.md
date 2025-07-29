## 🚀 Dropbox-like Cloud Storage System Design (30-Minute Interview Format)

### 🧭 30-Minute Interview Roadmap

| Time      | Section                       | Goal                                          |
| --------- | ----------------------------- | --------------------------------------------- |
| 0–5 min   | 📜 Requirements Clarification | Nail the problem scope and priorities         |
| 5–8 min   | 🔢 Capacity Estimation        | Show scale awareness with rough numbers       |
| 8–16 min  | 🏋️ High-Level Design         | Define system components, data flow, and APIs |
| 16–26 min | 🔬 Deep Dive                  | Explore one or two core components in detail  |
| 26–30 min | 📈 Bottlenecks & Scaling      | Address scalability, resilience, and wrap-up  |

---

### =⃣ Requirements Clarification (0–5 min)

**Ask These to Clarify Scope & Tradeoffs:**

* 🔢 *Scale*: "How many DAUs and files do we support?"
* ⏱️ *Latency*: "What's acceptable for sync? <5s?"
* 📁 *Data Types*: "Max file size? Binary vs. text?"
* ⚖️ *Tradeoffs*: "Consistency vs. availability?"
* 🔒 *Security*: "How strict are compliance and encryption?"

**Core Functional Requirements (Ranked):**

1. Upload / Download / Delete files
2. Real-time sync across devices
3. Sharing (links, access control)
4. File versioning
5. Search by name/content
6. User auth, quotas, profiles

**Key Non-Functional Constraints:**

* Resumable uploads (large files)
* Strong consistency for file content
* <5s sync latency across devices
* 99.9% availability
* Cost-efficient storage via deduplication
* Encryption at rest + transit

---

### 2⃣ Capacity Estimation (5–8 min)

**🧾 Quick Back-of-the-Envelope Math:**

| Metric             | Value                      |
| ------------------ | -------------------------- |
| **DAUs**           | 10M users                  |
| **Files/User**     | 500 files → 5B total files |
| **Avg File Size**  | 2MB → 10PB storage         |
| **Upload Ops**     | 50M/day → \~1.7K QPS peak  |
| **Download Ops**   | 200M/day → \~7K QPS peak   |
| **Sync Events**    | \~100K/sec globally        |
| **Bandwidth**      | 3.4GB/s up, 14GB/s down    |
| **Storage Growth** | +100TB/year                |
| **Metadata Size**  | \~500GB                    |

**📌 Insight:** Read-heavy (4:1), bursty usage, large scale

---

### 3⃣ High-Level Design (8–16 min)

#### 🧱 Architecture Overview

```
Client Apps ↔ CDN ↔ Load Balancer ↔ API Gateway
                                 ↓
                           Microservices:
                           ├── Auth
                           ├── Metadata
                           ├── Upload
                           ├── Sync
                           ├── Share
                           ├── Search
                           └── Notify
                                 ↓
                         Kafka/Event Queue
                                 ↓
     ┌───────────────┬───────────────┬───────────────┐
     │               │               │               │
 PostgreSQL        Redis         S3/Object Store  Elasticsearch
 (Metadata)        (Sessions)    (File Chunks)     (Search)
```

#### 🧹 Key APIs (Grouped)

**File APIs**

* `POST /files/upload` – Initiate upload
* `PUT /files/{fileId}/chunks/{chunkId}` – Upload chunk
* `POST /files/{fileId}/complete` – Finish upload
* `GET /files/{fileId}` – Download
* `DELETE /files/{fileId}` – Delete

**Sync & Versioning**

* `GET /sync/changes?since={ts}` – Delta sync
* `POST /sync/subscribe` – Real-time updates (WebSocket)
* `GET /files/{fileId}/versions` – Get version history

**Sharing**

* `POST /files/{id}/share` – Create share link
* `GET /shared/{token}` – Access shared file

**Search**

* `GET /search?q={query}` – File/folder search

---

### 4⃣ Deep Dive (16–26 min)

#### 🔹 Option A: Chunked File Upload + Deduplication

**Flow:**

1. Client computes SHA256
2. Check if checksum already exists → Deduplicate
3. If not, split into 4MB chunks
4. Upload chunks in parallel
5. Server merges → stores to S3 → updates metadata
6. Notify sync service

**Deduplication Pseudo-code:**

```python
def upload_file(content, user_id):
    checksum = sha256(content)
    if db.exists(checksum):
        link_metadata(user_id, checksum)
        return "Deduped"
    else:
        chunks = split_chunks(content)
        store_chunks(chunks)
        store_metadata(user_id, checksum)
        return "New Upload"
```

#### 🔹 Option B: Real-time Sync

**Architecture:**

* File change triggers Kafka event
* Sync Service picks it up
* Sends via WebSocket to clients
* Delta updates via `/sync/changes`

**Sync Strategies:**

* **Last-writer wins** (simple)
* **Vector clocks** (accurate, complex)
* **OT/CRDT** (if collaborative editing needed)

#### 🔹 Option C: Storage Lifecycle Management

* Hot: SSD-backed S3 (recent files)
* Warm: S3 Standard
* Cold: Glacier for archived/unused files
* Use lifecycle rules to auto-move tiers
* Deduplication at block level
* Compress text/doc files

---

### 5⃣ Scaling & Wrap-up (26–30 min)

#### 📉 Bottlenecks & Solutions

| Problem       | Solution                          |
| ------------- | --------------------------------- |
| Metadata DB   | Shard by user, Redis cache, CQRS  |
| Large uploads | Chunked uploads, CDN, retries     |
| Sync storms   | Batch updates, rate limiting      |
| Storage costs | Compression, deduplication, tiers |

#### 🛠️ Tech Stack Recap

| Layer       | Technology                |
| ----------- | ------------------------- |
| Frontend    | React, iOS/Android        |
| Backend     | Python/Java microservices |
| API Gateway | Kong / AWS API Gateway    |
| Queue       | Kafka                     |
| DB          | PostgreSQL, Redis         |
| Storage     | S3, Glacier               |
| Search      | Elasticsearch             |
| Monitoring  | Prometheus, Grafana       |

---

## 💪 Bonus: Follow-Up Q\&A Prep

| ❓ Question                  | ✅ Approach                            |
| --------------------------- | ------------------------------------- |
| Offline sync?               | Version vectors, merge logic          |
| Quota enforcement?          | Reject uploads, UI prompt             |
| Collaborative editing?      | OT / CRDT + WebRTC                    |
| Data integrity?             | SHA checksums + periodic audits       |
| File previews?              | Async pipeline + worker queue         |
| Disaster recovery?          | Cross-region replicas, backups        |
| Large file handling (>1GB)? | Streaming upload, CDN, resume support |

---

### ✅ Final Tips

* Start with **upload → sync → share** journey
* Emphasize **data durability**: "no file should ever be lost"
* Show **tradeoff thinking**: consistency vs availability
* Keep **mobile experience** in mind: offline, retries, compression
* Think **globally**: multi-region, CDN-aware, disaster-ready
