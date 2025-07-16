# Notification System Design - 30-Minute Interview Guide

## 🎯 Interview Timeline (30 minutes)
- **Requirements (5 min)**: Clarify functional/non-functional requirements
- **Capacity Estimation (3 min)**: Quick calculations with formulas
- **High-Level Design (8 min)**: Core services and API design
- **Deep Dive (10 min)**: Focus on 1-2 critical components
- **Scaling & Wrap-up (4 min)**: Discuss bottlenecks and solutions

---

## 1. Requirements Clarification (5 minutes)

### Key Questions to Ask:
- "What's the scale? How many users and notifications per day?"
- "What delivery channels do we need to support?"
- "Do we need real-time delivery or eventual consistency?"
- "What's the acceptable delivery latency?"
- "Do we need delivery guarantees and retry logic?"

### Functional Requirements (Priority Order):
1. **Event-triggered notifications** from upstream services
2. **Multi-channel delivery**: Push, SMS, Email, In-app
3. **User preferences**: Opt-in/out, channel preferences, quiet hours
4. **Template management**: Dynamic content with variables
5. **Retry logic and delivery guarantees**
6. **Dead letter handling** for failed deliveries
7. **Delivery tracking and analytics**

### Non-Functional Requirements:
- **Scalability**: 100M notifications/day, handle traffic spikes
- **Consistency**: Guarantee at-least-once delivery, avoid duplicates
- **Latency**: <5 seconds for real-time channels, <1 minute for email
- **Availability**: 99.9% uptime
- **Extensibility**: Easy to add new channels and templates
- **Observability**: Comprehensive logs, metrics, audit trails
- **Idempotency**: Safe retrying without duplicate deliveries

---

## 2. Capacity Estimation (3 minutes)

### 📊 Simple Formulas & Calculations

#### Traffic Estimation:
```
Daily Active Users: 10M
Notifications per user per day: 10
Total notifications: 100M/day

QPS Calculation:
- Base QPS: 100M notifications/day = ~1,200 QPS
- Peak multiplier: 5x = 6,000 QPS
- Channel breakdown:
  * Push: 50% = 3,000 QPS
  * Email: 30% = 1,800 QPS  
  * SMS: 15% = 900 QPS
  * In-app: 5% = 300 QPS
```

#### Storage Estimation:
```
Per notification: ~2KB (metadata + content)
Daily storage: 100M × 2KB = 200GB/day
Annual storage: 200GB × 365 = 73TB/year
With 3-year retention: ~220TB total

Template storage: ~10MB (hundreds of templates)
User preferences: 10M users × 1KB = 10GB
```

---

## Bottlenecks and Challenges

### Critical Challenges:
- **Event ingestion**: High-volume bursts from upstream services
- **Template rendering**: CPU-intensive with dynamic content
- **External provider APIs**: Rate limiting, throttling, timeouts
- **User preference lookup**: High QPS read operations
- **Delivery guarantees**: Handling failures and retries
- **Cost management**: SMS costs, provider rate limits

### Channel-Specific Challenges:
- **SMS**: High cost, carrier restrictions, character limits
- **Push**: Device offline, app uninstalled, platform differences
- **Email**: Spam filtering, deliverability, formatting
- **In-app**: Real-time updates, user session management

---

## 3. High-Level Design (8 minutes)

### Core Services Architecture:

```
Event Sources (User Service, Order Service, etc.)
        ↓
Event Ingestion Service (API Gateway)
        ↓
Message Broker (Kafka)
        ↓
Notification Orchestrator
        ↓
┌─────────────┬─────────────┬─────────────┐
│User Prefs   │Template     │Delivery     │
│Service      │Service      │Tracker      │
└─────────────┴─────────────┴─────────────┘
        ↓
Channel-Specific Queues (SQS/RabbitMQ)
        ↓
┌─────────────┬─────────────┬─────────────┬─────────────┐
│Push Worker  │SMS Worker   │Email Worker │In-app Worker│
└─────────────┴─────────────┴─────────────┴─────────────┘
        ↓
External Providers (Firebase, Twilio, SendGrid)
        ↓
Dead Letter Queue (Failed deliveries)
```

### Communication Flow:
1. **Event Source** → generates notification event
2. **Event Ingestion** → validates and enriches event
3. **Message Broker** → reliable event distribution
4. **Notification Orchestrator** → processes event, fetches preferences
5. **Template Service** → renders notification content
6. **Channel Queues** → distribute to appropriate workers
7. **Channel Workers** → deliver via external providers
8. **Delivery Tracker** → records delivery status

### Key APIs:

#### Send Notification
```http
POST /api/v1/notifications
{
  "event_id": "order_confirmed_123",
  "user_id": "user_456",
  "template_id": "order_confirmation",
  "channels": ["push", "email"],
  "data": {
    "order_id": "ORD-789",
    "total_amount": "$25.99"
  },
  "idempotency_key": "uuid-12345"
}
```

#### Get Delivery Status
```http
GET /api/v1/notifications/{notification_id}/status
Response:
{
  "notification_id": "notif_123",
  "status": "delivered",
  "channels": [
    {
      "type": "push",
      "status": "delivered",
      "delivered_at": "2024-01-15T10:30:00Z"
    },
    {
      "type": "email", 
      "status": "pending",
      "attempts": 1
    }
  ]
}
```

#### User Preferences
```http
PUT /api/v1/users/{user_id}/preferences
{
  "channels": {
    "push": true,
    "email": true,
    "sms": false
  },
  "quiet_hours": {
    "start": "22:00",
    "end": "08:00",
    "timezone": "PST"
  },
  "categories": {
    "marketing": false,
    "transactional": true
  }
}
```

### Database Design:

```sql
-- Notifications table
CREATE TABLE notifications (
    id UUID PRIMARY KEY,
    user_id BIGINT NOT NULL,
    event_id VARCHAR(255),
    template_id VARCHAR(100),
    content JSONB,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_created (user_id, created_at),
    INDEX idx_event_id (event_id)
);

-- Delivery attempts table
CREATE TABLE delivery_attempts (
    id UUID PRIMARY KEY,
    notification_id UUID REFERENCES notifications(id),
    channel VARCHAR(20) NOT NULL,
    provider VARCHAR(50),
    status VARCHAR(20), -- sent, delivered, failed
    error_message TEXT,
    attempted_at TIMESTAMP DEFAULT NOW(),
    delivered_at TIMESTAMP,
    INDEX idx_notification_channel (notification_id, channel)
);

-- User preferences table
CREATE TABLE user_preferences (
    user_id BIGINT PRIMARY KEY,
    preferences JSONB NOT NULL,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Templates table
CREATE TABLE templates (
    id VARCHAR(100) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    content JSONB NOT NULL, -- channel-specific templates
    variables JSONB, -- expected variables
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 4. Deep Dive Topics (10 minutes)

### A. Delivery Guarantees & Retry Logic

#### At-Least-Once Delivery Strategy:
```python
class NotificationDeliveryService:
    def deliver_notification(self, notification_id, channel, max_retries=3):
        attempt = 0
        while attempt < max_retries:
            try:
                # Record attempt
                self.record_attempt(notification_id, channel, attempt)
                
                # Send notification
                result = self.send_via_provider(notification_id, channel)
                
                if result.success:
                    self.mark_delivered(notification_id, channel)
                    return True
                    
            except Exception as e:
                self.log_error(notification_id, channel, attempt, e)
                
            attempt += 1
            self.exponential_backoff(attempt)
            
        # Send to dead letter queue
        self.send_to_dlq(notification_id, channel)
        return False
```

#### Idempotency Implementation:
```python
def process_notification(self, event):
    idempotency_key = event.get('idempotency_key')
    
    # Check if already processed
    existing = self.get_by_idempotency_key(idempotency_key)
    if existing:
        return existing  # Return previous result
    
    # Process new notification
    notification = self.create_notification(event)
    self.store_with_idempotency_key(notification, idempotency_key)
    
    return notification
```

### B. Template Rendering & Content Management

#### Dynamic Template System:
```json
{
  "template_id": "order_confirmation",
  "channels": {
    "push": {
      "title": "Order Confirmed! 🎉",
      "body": "Your order #{{order_id}} for {{total_amount}} has been confirmed."
    },
    "email": {
      "subject": "Order Confirmation - {{order_id}}",
      "html": "<h1>Thanks for your order!</h1><p>Order: {{order_id}}</p>",
      "text": "Thanks for your order! Order: {{order_id}}"
    },
    "sms": {
      "body": "Order {{order_id}} confirmed for {{total_amount}}. Track: {{tracking_url}}"
    }
  }
}
```

#### Template Rendering Service:
```python
class TemplateRenderer:
    def render(self, template_id, channel, data):
        template = self.get_template(template_id)
        channel_template = template['channels'][channel]
        
        # Use templating engine (Jinja2, Handlebars)
        rendered = {}
        for key, value in channel_template.items():
            rendered[key] = self.template_engine.render(value, data)
            
        return rendered
```

### C. Channel-Specific Considerations

#### Push Notifications:
```python
class PushNotificationWorker:
    def send_push(self, notification):
        # Handle different platforms
        if notification.device_type == 'ios':
            return self.send_apns(notification)
        elif notification.device_type == 'android':
            return self.send_fcm(notification)
        
        # Handle device offline, app uninstalled
        if result.error == 'device_not_registered':
            self.mark_device_inactive(notification.device_token)
```

#### SMS with Rate Limiting:
```python
class SMSWorker:
    def __init__(self):
        self.rate_limiter = RateLimiter(100, 60)  # 100 SMS per minute
        
    def send_sms(self, notification):
        if not self.rate_limiter.allow():
            # Queue for later
            self.delay_notification(notification, delay=60)
            return
            
        return self.twilio_client.send(notification)
```

---

## 5. Scaling & Wrap-up (4 minutes)

### Strategic Tech & Infrastructure Decisions

#### Technology Stack:
- **Message Broker**: Kafka for event streaming, high throughput
- **Databases**: PostgreSQL for transactional data, Redis for caching
- **Queue System**: AWS SQS/RabbitMQ for channel-specific queues
- **Template Engine**: Handlebars.js for dynamic content
- **External Providers**: Firebase (Push), Twilio (SMS), SendGrid (Email)
- **Deployment**: Kubernetes for auto-scaling, AWS Lambda for workers
- **Monitoring**: Datadog for metrics, ELK stack for logs

#### Scaling Strategies:

**Horizontal Scaling:**
- Partition Kafka topics by user_id
- Scale workers independently based on channel load
- Use read replicas for user preferences

**Caching Strategy:**
- User preferences: Redis cache with 1-hour TTL
- Templates: In-memory cache with CDN
- Delivery status: Redis for recent deliveries

**Performance Optimizations:**
- Batch processing for email notifications
- Connection pooling for external APIs
- Async processing for non-real-time channels

### Monitoring & Observability:

#### Key Metrics:
- **Throughput**: Notifications/second per channel
- **Latency**: P95 delivery time per channel
- **Success Rate**: Delivery success percentage
- **Error Rate**: Failed deliveries by error type
- **Cost**: SMS/push notification costs

#### Alerting:
- High error rates (>5%)
- Unusual traffic spikes
- External provider outages
- Queue depth thresholds

### Final Architecture:

```
Event Sources
      ↓
[Event Ingestion API] → [Kafka] → [Notification Orchestrator]
      ↓                              ↓
[User Preferences Cache] ← [Template Service]
      ↓                              ↓
[Channel Queues] → [Workers] → [External Providers]
      ↓                              ↓
[Delivery Tracker] ← [Dead Letter Queue]
      ↓
[Analytics & Monitoring]
```

---

## Common Interview Follow-ups & Tips

### Expected Questions:
1. **"How do you handle a celebrity user with millions of followers?"**
   - Use fan-out limits, queue prioritization, rate limiting

2. **"What if an external provider goes down?"**
   - Circuit breaker pattern, fallback providers, graceful degradation

3. **"How do you prevent spam and abuse?"**
   - Rate limiting per user, content filtering, user reputation

4. **"How do you handle different time zones?"**
   - Store user timezone, respect quiet hours, schedule delivery

### Interview Success Tips:
1. **Start with MVP**: Basic notification → Add complexity gradually
2. **Show trade-offs**: Discuss consistency vs. performance
3. **Think about costs**: SMS expensive, push cheap
4. **Consider failures**: Network issues, provider limits
5. **Discuss monitoring**: How to detect and fix issues
6. **Be practical**: Real-world constraints and solutions

**Remember**: Focus on demonstrating systematic thinking, scalability awareness, and practical engineering decisions rather than memorizing specific technologies.