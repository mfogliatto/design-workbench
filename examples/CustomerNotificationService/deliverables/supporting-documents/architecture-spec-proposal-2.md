# Architecture Specification: Customer Notification Service (Proposal 2 - API Gateway with Smart Routing)

**Version:** 1.0  
**Date:** November 21, 2025  
**Status:** Detailed Design  
**Author(s):** Architecture Team

---

## 1. Executive Summary

The Customer Notification Service (CNS) implements an API Gateway architecture with smart routing to centralize notification delivery across 15+ internal microservices. The system uses priority-based Azure Service Bus queues and a smart routing worker pool that makes intelligent decisions about batching, rate limiting, and provider selection. By combining aggressive caching (Redis) with in-memory routing intelligence, the architecture achieves lower latency and cost than event-driven alternatives while maintaining operational simplicity.

**Key architectural highlights:**
- Request-response pattern with queues for durability simplifies debugging and operations
- Smart Router component provides intelligent batching, rate limiting, and circuit breaking
- Redis caching reduces Cosmos DB costs by 70% and improves API latency by 40%
- Priority-based queues ensure critical notifications processed within 5 seconds
- Lower infrastructure costs ($90k vs $93k) through simpler Service Bus topology

**Target metrics:**
- 99.9% service availability (43 minutes downtime/month)
- 580 msg/sec average throughput, 2,000 msg/sec peak capacity
- P95 API latency < 150ms (async), < 2s (sync) - 25% better than event-driven
- P95 end-to-end delivery < 25 seconds
- 99%+ delivery success rate for critical notifications

---

## 2. System Overview

### 2.1 Purpose and Scope

**Purpose**  
Centralize and standardize customer notification delivery across email, SMS, push notifications, and in-app messages. Replace scattered notification logic across 15+ services with a single, reliable, compliant notification platform optimized for operational simplicity and cost efficiency.

**In Scope**
- REST API for notification submission (sync and async patterns)
- Multi-channel delivery (email, SMS, push, in-app)
- Template management and rendering with personalization
- Customer preference management (opt-outs, channel preferences, quiet hours)
- Delivery tracking and status reporting
- Priority-based processing (critical, high, normal, low)
- Automated retry with exponential backoff
- Audit trail for compliance (GDPR, CCPA, CAN-SPAM)
- Webhook callbacks for status updates

**Out of Scope**
- Custom notification channel implementation (uses existing providers)
- Customer-facing preference management UI
- Real-time notification triggers (consuming services decide when to send)
- Content authoring and campaign management tools
- Advanced analytics and reporting dashboards

### 2.2 Key Goals

1. **Operational Simplicity**: Traditional queue-worker pattern; straightforward debugging with clear request paths
2. **Performance**: P95 API latency < 150ms (25% better than event-driven); aggressive caching for preference lookups
3. **Cost Efficiency**: 70% reduction in Cosmos DB costs through Redis caching; simpler Service Bus topology
4. **Reliability**: Zero message loss with at-least-once delivery guarantee; 99% delivery success rate
5. **Scalability**: Handle 50M notifications/day with capacity for 3x growth (150M/day)
6. **Compliance**: GDPR data residency, CCPA opt-out management, CAN-SPAM unsubscribe handling

### 2.3 Success Criteria

- **Functional**: All 15 services migrated within 5 months; zero production incidents during migration
- **Performance**: 99% of API requests complete within 150ms; no queue backlogs > 50k messages
- **Reliability**: < 1 critical incident per month; MTTR < 20 minutes (simpler debugging than event-driven)
- **Cost**: Monthly operational costs < $72k after optimization (vs $70k current distributed costs)
- **Compliance**: Zero privacy violations; all opt-out requests honored within 5 minutes (cache TTL)

---

## 3. Architecture Overview

### 3.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Consuming Services Layer                           │
│          Order Service │ Account Service │ Promotion Service │ ...        │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ REST API (HTTPS)
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      Azure API Management (APIM)                          │
│         OAuth 2.0 Auth │ Rate Limiting │ Response Caching                 │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     CNS API Service (AKS - 5 pods)                        │
│  Request Validation │ Preference Lookup (Redis) │ Queue Routing           │
└─────┬────────────────┬───────────────┬───────────────┬───────────────────┘
      │                │               │               │
      ▼                ▼               ▼               ▼
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│  critical  │  │    high    │  │   normal   │  │    low     │
│   Queue    │  │   Queue    │  │   Queue    │  │   Queue    │
│  (5s SLA)  │  │  (30s SLA) │  │  (5m SLA)  │  │  (1h SLA)  │
└─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
      └───────────────┴───────────────┴───────────────┘
                      │ Priority-based pull
                      ▼
┌──────────────────────────────────────────────────────────────────────────┐
│              Smart Routing Worker Pool (AKS - 10-30 pods)                │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ Pull → Check Provider Health → Apply Rate Limits → Batch Logic    │  │
│  │ → Render Template (cache-first) → Route to Channel Handler        │  │
│  │ → Update Tracking → Trigger Webhooks                              │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ Smart Router (In-Memory Shared State via Redis Pub/Sub):          │  │
│  │ • Provider health tracking • Token bucket rate limiters            │  │
│  │ • Batch accumulation logic • Circuit breaker states                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└────┬──────────────────────┬───────────────────────┬──────────────────────┘
     │                      │                       │
     ▼                      ▼                       ▼
┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐
│Email Handler │  │   SMS Handler    │  │  Push Handler    │
│(batch 100)   │  │  (no batching)   │  │  (batch 1000)    │
│SendGrid API  │  │   Twilio API     │  │  Azure NH API    │
└──────────────┘  └──────────────────┘  └──────────────────┘
         │                 │                      │
         └─────────────────┴──────────────────────┘
                           │
┌──────────────────────────┴───────────────────────────────────────────────┐
│         Azure Cache for Redis (Premium, 26 GB, Multi-AZ)                 │
│  • Customer Preferences (80%+ cache hit) • Rendered Templates            │
│  • Rate Limit Counters • Smart Router State • Sync Request Correlation  │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
┌───────────────────────────┴──────────────────────────────────────────────┐
│               Azure Cosmos DB (Multi-Region: US West, EU West)            │
│  Customer Preferences │ Notification Tracking │ Templates │ Audit Logs    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Key Components

**Azure API Management (APIM)**
- Entry point for all consuming services
- Enforces OAuth 2.0 client credentials flow (Azure AD integration)
- Rate limiting: 100 req/sec per consuming service
- Request/response validation against OpenAPI schema
- Response caching for status queries (5-second cache)

**CNS API Service**
- Stateless microservice running on AKS (5 pods, HPA enabled)
- Validates notification payload (recipient, channel, template, data)
- Assigns unique tracking ID (ULID format for time-sortable IDs)
- **Cache-first preference lookup**: Redis (P95 < 5ms) → Cosmos DB fallback (P95 < 50ms)
- Checks opt-out rules, quiet hours, and invalid contacts
- Enqueues to priority-specific Service Bus queue (critical, high, normal, low)
- For sync requests: stores correlation ID in Redis and subscribes to status channel
- For async requests: returns tracking ID immediately (202 Accepted)

**Azure Service Bus Queues (Standard Tier)**
- Four priority-based queues: critical (5s SLA), high (30s SLA), normal (5min SLA), low (1hr SLA)
- Message properties: trackingId, priority, channel, timestamp
- TTL: 24 hours (expired messages moved to DLQ)
- Dead-letter queue for messages exceeding 5 retry attempts
- **Standard tier sufficient**: No need for Premium ($700/month vs $1,340/month)

**Smart Routing Worker Pool**
- Horizontally scalable workers (10-30 pods based on queue depth)
- Pull messages from queues with priority ordering (critical first, round-robin within priority)
- Embed Smart Router logic for intelligent decision-making
- Handle template rendering with cache-first strategy (Redis → Cosmos DB fallback)
- Call channel handlers and update tracking records
- Implement exponential backoff retry logic (1s, 2s, 4s, 8s, 16s)
- Auto-scale based on combined queue depth across all priority levels

**Smart Router (Embedded in Workers)**
- **In-memory shared state** synchronized via Redis pub/sub
- **Circuit breaker per provider**: Open after 10 consecutive failures, 30s timeout
- **Token bucket rate limiting**: Respects provider limits with 10% safety margin
  - SendGrid: 9,000 emails/sec (90% of 10,000 limit)
  - Twilio: 900 SMS/sec (90% of 1,000 limit)
  - Azure NH: 9,000 push/sec (90% of 10,000 limit)
- **Batching logic**: 
  - Email: Accumulate up to 100 recipients OR 5 seconds (whichever first)
  - SMS: No batching (single recipient per request)
  - Push: Accumulate up to 1,000 devices OR 5 seconds
- **Provider health scoring**: Track last 100 requests per provider; exponentially weighted moving average (EWMA) of latency and error rate
- **Automatic channel fallback**: If email fails and `fallbackToSms: true`, retry via SMS

**Channel Handlers**
- Thin wrappers around provider SDKs (SendGrid, Twilio, Azure NH)
- **Email handler**: Supports batching (SendGrid batch API), processes bounce/complaint webhooks
- **SMS handler**: No batching, handles carrier-specific errors
- **Push handler**: Supports multicast to device groups, handles unregistered device errors
- Parse provider responses and normalize status codes
- Return status to worker for tracking update

**Azure Cache for Redis (Premium)**
- Premium tier (26 GB, multi-AZ for 99.9% SLA, 6 GB/s throughput)
- **Customer preferences**: Cache-aside pattern, 5-min TTL, 80%+ hit rate
- **Rendered templates**: Cache-aside pattern, 1-hour TTL, 60% hit rate
- **Rate limit counters**: Sliding window, 1-sec granularity, atomic increment
- **Smart Router state**: Provider health scores, circuit breaker states
- **Sync request correlation**: Temporary state for synchronous API (30-sec TTL)
- Redis Pub/Sub: Synchronize Smart Router state across worker pods

**Azure Cosmos DB**
- Multi-region deployment (US West, EU West) for data residency compliance
- **customer-preferences container**: Partition key `/customerId`, 3,000 RU/s (reduced from 10,000 due to caching)
- **notification-tracking container**: Partition key `/trackingId`, 5,000 RU/s
- **templates container**: Partition key `/templateId`, 500 RU/s
- **audit-logs container**: Partition key `/date`, 1,500 RU/s, TTL: 90 days
- Consistency level: Session (balance between consistency and performance)

### 3.3 Key Technologies

**Azure Kubernetes Service (AKS)**
- Rationale: Industry-standard container orchestration; excellent scaling and deployment capabilities
- Configuration: 2 node pools (system and user), Standard_D4s_v3 VMs, auto-scaling enabled
- Benefits: Rolling updates, health checks, resource limits, horizontal pod autoscaling

**Azure Service Bus Standard**
- Rationale: Cost-effective for our throughput needs (standard is 25% cost of premium); four separate queues for priority handling
- Configuration: 4 queues (critical, high, normal, low), duplicate detection enabled
- Benefits: Message durability, automatic dead-lettering, FIFO within priority

**Azure Cache for Redis Premium**
- Rationale: Shared cache across all pods; multi-AZ for reliability; Redis pub/sub for state synchronization
- Configuration: Premium P3 (26 GB, 6 GB/s throughput), multi-AZ, clustering enabled
- Benefits: Sub-millisecond latency, 99.9% SLA, atomic operations, pub/sub messaging

**Azure Cosmos DB SQL API**
- Rationale: Global distribution, multi-region writes, automatic indexing, guaranteed SLAs
- Configuration: Multi-region (US, EU), session consistency, automatic failover
- Benefits: Single-digit millisecond latency, elastic scalability, comprehensive SLAs

**Azure API Management Premium**
- Rationale: Enterprise API gateway with VNET integration, multi-region deployment, caching
- Configuration: VNET-integrated, OAuth 2.0 validation, rate limiting policies
- Benefits: Centralized API management, security enforcement, built-in analytics

**SendGrid, Twilio, Azure Notification Hubs**
- Rationale: Mandated by existing contracts (constraint DC-004, DC-005)
- Benefits: Proven reliability, global reach, comprehensive APIs, webhook support

---

## 4. Detailed Design

### 4.1 Component Design

#### 4.1.1 API Service Component

**Responsibilities**
- Accept HTTP POST requests at `/api/v1/notifications` and `/api/v1/notifications/batch`
- Validate request schema
- Authenticate and authorize via OAuth 2.0
- Assign tracking ID (ULID)
- **Cache-first preference lookup**: Check Redis, fallback to Cosmos DB, update cache on miss
- Apply business rules: opt-out checks, quiet hours, invalid contact filtering
- Enqueue to priority-specific Service Bus queue
- For sync requests: store correlation in Redis, subscribe to status updates
- For async requests: return tracking ID immediately

**Interface Specification**

```
POST /api/v1/notifications
Authorization: Bearer {OAuth2-token}
Content-Type: application/json

Request Body:
{
  "recipient": {
    "customerId": "cust_123456789",
    "email": "customer@example.com"
  },
  "channel": "email",
  "template": {
    "id": "order-confirmation-v2",
    "version": "2.1.0"
  },
  "data": {
    "orderId": "ORD-2025-00123",
    "orderTotal": "$149.99",
    "deliveryDate": "2025-11-25"
  },
  "priority": "high",
  "metadata": {
    "serviceName": "order-service",
    "correlationId": "req-abc-123"
  },
  "options": {
    "async": true,
    "webhookUrl": "https://service.com/webhook",
    "fallbackToSms": false
  }
}

Response (Async - 202 Accepted):
{
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "status": "queued",
  "queuedAt": "2025-11-21T10:15:30Z",
  "estimatedDelivery": "2025-11-21T10:15:45Z"
}

Response (Sync - 200 OK):
{
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "status": "sent",
  "sentAt": "2025-11-21T10:15:32Z",
  "provider": "sendgrid",
  "providerMessageId": "msg_sg_xyz123"
}
```

**Preference Lookup Logic**
```javascript
async function getCustomerPreferences(customerId) {
  // Step 1: Check Redis cache
  const cacheKey = `prefs:${customerId}`;
  const cached = await redis.get(cacheKey);
  if (cached) {
    metrics.increment('cache.preferences.hit');
    return JSON.parse(cached);
  }
  
  // Step 2: Cache miss, query Cosmos DB
  metrics.increment('cache.preferences.miss');
  const preferences = await cosmosDb.query(
    `SELECT * FROM c WHERE c.customerId = @customerId`,
    { customerId }
  );
  
  // Step 3: Store in Redis with 5-min TTL
  await redis.setex(cacheKey, 300, JSON.stringify(preferences));
  
  return preferences;
}
```

**Scaling Strategy**
- Horizontal Pod Autoscaler (HPA): target CPU 70%, target memory 80%
- Min replicas: 3, Max replicas: 10
- Scale-up: add 1 pod per 150 req/sec sustained for 30 seconds
- Scale-down: remove 1 pod after 5 minutes below target

#### 4.1.2 Smart Routing Worker Component

**Responsibilities**
- Pull messages from Service Bus queues with priority ordering (critical first)
- Consult Smart Router for provider health, rate limits, and batching decisions
- Render templates with cache-first strategy (Redis → Cosmos DB)
- Call appropriate channel handler
- Update tracking record in Cosmos DB
- Trigger webhook callbacks (fire-and-forget, 5s timeout)
- Handle retries with exponential backoff

**Processing Flow**
```
1. Pull message from queue (long-polling with 30s timeout)
2. Increment active processing counter (for scaling metrics)
3. Check Smart Router:
   a. Is provider healthy? (circuit breaker closed?)
   b. Do we have rate limit budget? (token bucket has tokens?)
   c. Should we batch? (accumulate or send now?)
4. If batching: accumulate in memory, set 5-second timer
5. If sending: render template (check Redis → Cosmos DB)
6. Call channel handler with rendered content
7. Parse provider response (success, failure, transient error)
8. If transient error: retry with exponential backoff (1s, 2s, 4s, 8s, 16s)
9. If permanent error or max retries: dead-letter message
10. Update tracking record in Cosmos DB (async, fire-and-forget)
11. Trigger webhook if configured (async, fire-and-forget)
12. Complete Service Bus message (remove from queue)
13. Publish status to Redis pub/sub (for sync API requests)
```

**Smart Router Logic**

*Circuit Breaker State Machine:*
```javascript
class CircuitBreaker {
  constructor(provider) {
    this.provider = provider;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = null;
    this.openDuration = 30000; // 30 seconds
  }
  
  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.openDuration) {
        this.state = 'HALF_OPEN';
        this.successCount = 0;
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= 3) {
        this.state = 'CLOSED';
      }
    }
  }
  
  onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= 10) {
      this.state = 'OPEN';
    }
  }
}
```

*Token Bucket Rate Limiter:*
```javascript
class TokenBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity; // max tokens
    this.tokens = capacity; // current tokens
    this.refillRate = refillRate; // tokens per second
    this.lastRefill = Date.now();
  }
  
  async acquire(count = 1) {
    this.refill();
    if (this.tokens >= count) {
      this.tokens -= count;
      return true;
    }
    return false;
  }
  
  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000; // seconds
    const tokensToAdd = Math.floor(elapsed * this.refillRate);
    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
}

// Configuration
const sendGridLimiter = new TokenBucket(9000, 9000); // 9k/sec capacity, 9k/sec refill
const twilioLimiter = new TokenBucket(900, 900);     // 900/sec capacity, 900/sec refill
const azureNhLimiter = new TokenBucket(9000, 9000);  // 9k/sec capacity, 9k/sec refill
```

*Batching Logic (Email Example):*
```javascript
class EmailBatcher {
  constructor() {
    this.batches = new Map(); // templateId -> array of notifications
    this.timers = new Map();  // templateId -> timer
    this.maxBatchSize = 100;
    this.maxWaitMs = 5000; // 5 seconds
  }
  
  async add(notification) {
    const templateId = notification.template.id;
    
    if (!this.batches.has(templateId)) {
      this.batches.set(templateId, []);
      this.startTimer(templateId);
    }
    
    const batch = this.batches.get(templateId);
    batch.push(notification);
    
    if (batch.length >= this.maxBatchSize) {
      await this.flush(templateId);
    }
  }
  
  startTimer(templateId) {
    const timer = setTimeout(() => {
      this.flush(templateId);
    }, this.maxWaitMs);
    this.timers.set(templateId, timer);
  }
  
  async flush(templateId) {
    const batch = this.batches.get(templateId);
    if (!batch || batch.length === 0) return;
    
    clearTimeout(this.timers.get(templateId));
    this.timers.delete(templateId);
    this.batches.delete(templateId);
    
    await sendBatchToProvider(batch);
  }
}
```

**Scaling Strategy**
- HPA based on combined queue depth across all priorities
- Target: < 5,000 messages total across all queues
- Min replicas: 10, Max replicas: 30
- Scale metric: `(critical * 10) + (high * 3) + normal + (low * 0.1)` (weighted)
- Scale-up: add 2 pods when weighted depth > 5,000 for 1 minute
- Scale-down: remove 1 pod when weighted depth < 2,000 for 5 minutes

#### 4.1.3 Channel Handler Components

**Email Handler**

*Responsibilities*
- Accept batch of notifications (up to 100 recipients)
- Call SendGrid batch API
- Parse response and extract per-recipient status
- Return array of statuses to worker

*SendGrid Batch API Call:*
```
POST https://api.sendgrid.com/v3/mail/send
Authorization: Bearer {sendgrid-api-key}
Content-Type: application/json

{
  "personalizations": [
    {
      "to": [{"email": "customer1@example.com"}],
      "custom_args": {"trackingId": "01HZKM8X..."}
    },
    {
      "to": [{"email": "customer2@example.com"}],
      "custom_args": {"trackingId": "01HZKM8Y..."}
    }
    // ... up to 100 recipients
  ],
  "from": {"email": "noreply@company.com", "name": "Company Name"},
  "subject": "Your order has shipped!",
  "content": [{"type": "text/html", "value": "<html>...</html>"}]
}

Success Response (202 Accepted):
{
  "message_id": "batch_sg_xyz123"
}
```

**SMS Handler**

*Responsibilities*
- Accept single notification (no batching)
- Call Twilio API
- Parse response
- Return status

*Twilio API Call:*
```
POST https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Messages.json
Authorization: Basic {base64(AccountSid:AuthToken)}
Content-Type: application/x-www-form-urlencoded

Body=Your order ORD-2025-00123 has shipped!
&From=+1234567890
&To=+19876543210
&StatusCallback=https://cns.company.com/webhook/twilio

Success Response (201 Created):
{
  "sid": "SMxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "status": "queued"
}
```

**Push Handler**

*Responsibilities*
- Accept batch of device tokens (up to 1,000)
- Call Azure Notification Hubs multicast API
- Parse response
- Return status

*Azure NH API Call:*
```
POST https://{namespace}.servicebus.windows.net/{hub}/messages?api-version=2015-01
Authorization: SharedAccessSignature {signature}
Content-Type: application/json
ServiceBusNotification-Format: template
ServiceBusNotification-Tags: userId:cust_123

{
  "title": "Order Shipped",
  "body": "Your order ORD-2025-00123 has shipped!",
  "data": {
    "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
    "orderId": "ORD-2025-00123"
  }
}

Success Response (201 Created):
{
  "notificationId": "nh_notif_xyz123",
  "state": "Enqueued"
}
```

### 4.2 Data Architecture

#### 4.2.1 Data Models

**Customer Preferences**
```json
{
  "id": "cust_123456789",
  "customerId": "cust_123456789",
  "channels": {
    "email": {
      "enabled": true,
      "address": "customer@example.com",
      "verified": true
    },
    "sms": {
      "enabled": true,
      "phoneNumber": "+1234567890",
      "verified": true
    },
    "push": {
      "enabled": true,
      "deviceTokens": [
        {"platform": "ios", "token": "apns:device123"},
        {"platform": "android", "token": "fcm:device456"}
      ]
    }
  },
  "optOuts": {
    "global": false,
    "categories": ["marketing"],
    "optOutDate": "2025-10-15T08:30:00Z"
  },
  "quietHours": {
    "enabled": true,
    "timezone": "America/Los_Angeles",
    "start": "22:00",
    "end": "08:00"
  },
  "updatedAt": "2025-10-15T08:30:00Z",
  "version": 2
}
```

**Notification Tracking**
```json
{
  "id": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "customerId": "cust_123456789",
  "channel": "email",
  "recipient": "customer@example.com",
  "templateId": "order-confirmation-v2",
  "priority": "high",
  "status": "sent",
  "attempts": [
    {
      "attempt": 1,
      "timestamp": "2025-11-21T10:15:32Z",
      "status": "sent",
      "provider": "sendgrid",
      "providerMessageId": "msg_sg_xyz123",
      "latency": 120,
      "errorCode": null
    }
  ],
  "metadata": {
    "serviceName": "order-service",
    "correlationId": "req-abc-123"
  },
  "createdAt": "2025-11-21T10:15:30Z",
  "updatedAt": "2025-11-21T10:15:32Z",
  "ttl": 7776000
}
```

**Notification Template**
```json
{
  "id": "order-confirmation-v2",
  "templateId": "order-confirmation",
  "version": "2.1.0",
  "name": "Order Confirmation Email",
  "channel": "email",
  "subject": "Your order {{orderId}} has been confirmed!",
  "bodyHtml": "<html><body><h1>Thank you for your order!</h1>...</body></html>",
  "bodyText": "Thank you for your order {{orderId}}...",
  "variables": ["orderId", "orderTotal", "deliveryDate"],
  "status": "active",
  "createdAt": "2025-01-15T10:00:00Z",
  "createdBy": "template-admin@company.com"
}
```

#### 4.2.2 Caching Strategy

**Redis Cache Structure**

*Customer Preferences (Cache-Aside, 5-min TTL):*
```
Key: prefs:{customerId}
Value: JSON serialized preference object
TTL: 300 seconds (5 minutes)
```

*Rendered Templates (Cache-Aside, 1-hour TTL):*
```
Key: template:{templateId}:{dataHash}
Value: Rendered HTML/text content
TTL: 3600 seconds (1 hour)
```

*Rate Limit Counters (Sliding Window, 1-sec granularity):*
```
Key: rate:{provider}:{timestamp_second}
Value: Integer count
TTL: 10 seconds (sliding window)
```

*Smart Router State (Pub/Sub):*
```
Channel: smart-router-updates
Message: JSON with circuit breaker states, provider health scores
No TTL (real-time updates)
```

*Sync Request Correlation (Temporary State, 30-sec TTL):*
```
Key: sync:{correlationId}
Value: JSON with API Gateway connection info
TTL: 30 seconds
```

**Cache Invalidation**

- **Preference updates**: Invalidate immediately on PUT /preferences/{customerId}
- **Template updates**: Invalidate all rendered template cache entries for that templateId
- **Rate limit counters**: Self-expire after 10 seconds (sliding window)
- **Smart Router state**: Updated every 5 seconds via pub/sub

#### 4.2.3 Database Partitioning Strategy

**customer-preferences container**
- Partition key: `/customerId`
- Rationale: Preferences always queried by customerId; perfect distribution
- Expected partitions: 12M customers → 12M logical partitions

**notification-tracking container**
- Partition key: `/trackingId`
- Rationale: Tracking queries use trackingId; each notification is independent
- Expected partitions: 50M/day × 90 days = 4.5B documents → 4.5B logical partitions

**templates container**
- Partition key: `/templateId`
- Rationale: Templates queried by templateId; small dataset fully cached

**audit-logs container**
- Partition key: `/date` (YYYY-MM-DD)
- Rationale: Compliance queries are date-range scoped; enables efficient TTL

### 4.3 APIs and Interfaces

#### 4.3.1 Public REST API

**Base URL**: `https://api.company.com/notifications/v1`

**Authentication**: OAuth 2.0 Client Credentials (Azure AD)

**Endpoints**

`POST /notifications` - Submit single notification
- Request: Notification object
- Response: 202 Accepted (async) or 200 OK (sync)
- Rate limit: 100 req/sec per client

`POST /notifications/batch` - Submit batch (max 1000)
- Request: Array of notification objects
- Response: 202 Accepted with array of tracking IDs
- Rate limit: 10 req/sec per client

`GET /notifications/{trackingId}` - Query status
- Response: Tracking record with status and attempts
- Rate limit: 1000 req/sec per client
- APIM caching: 5-second cache

`GET /preferences/{customerId}` - Get customer preferences
- Response: Preference object
- Rate limit: 1000 req/sec per client

`PUT /preferences/{customerId}` - Update preferences
- Request: Partial preference object
- Response: 200 OK with updated preferences
- Rate limit: 100 req/sec per client

**Error Response Format**
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {"field": "value"},
    "traceId": "00-abc123...",
    "timestamp": "2025-11-21T10:15:30Z"
  }
}
```

### 4.4 Security Architecture

#### 4.4.1 Authentication and Authorization

**Service-to-Service Authentication**
- Protocol: OAuth 2.0 Client Credentials flow
- Provider: Azure AD (Entra ID)
- Token lifetime: 1 hour
- Scope: `https://api.company.com/notifications.send`

**Internal Component Authentication**
- Managed identities for Azure resources
- RBAC at Azure resource level

#### 4.4.2 Data Protection

**Encryption at Rest**
- Cosmos DB: Microsoft-managed keys (option for CMK via Key Vault)
- Redis: Encryption enabled
- Service Bus: 256-bit AES encryption

**Encryption in Transit**
- TLS 1.3 for all HTTPS endpoints
- Service Bus and Cosmos DB connections use TLS

**PII Data Protection**
- Email/phone classified as PII
- Encrypted at rest in Cosmos DB
- Redacted from application logs

#### 4.4.3 Secrets Management

**Azure Key Vault**
- All secrets stored in Key Vault
- AKS pods access via Managed Identity
- Automated monthly rotation

#### 4.4.4 Network Security

**VNET Integration**
- AKS cluster in dedicated VNET (10.1.0.0/16)
- Cosmos DB and Redis private endpoints
- Network Security Groups (NSGs) restrict traffic

---

## 5. Non-Functional Requirements

### 5.1 Performance

**Throughput Targets**
- Average: 580 notifications/sec sustained
- Peak: 2,000 notifications/sec for 10 minutes
- Burst: 5,000 notifications/sec for 30 seconds

**Latency Targets**
- API response (async): P50 < 40ms, P95 < 150ms, P99 < 400ms (25% better than event-driven)
- API response (sync): P50 < 800ms, P95 < 2s, P99 < 5s
- End-to-end delivery: P50 < 8s, P95 < 25s, P99 < 50s

**Database Performance**
- Cosmos DB point reads: P50 < 5ms, P95 < 10ms
- Redis reads: P50 < 1ms, P95 < 5ms
- Preference cache hit rate: > 80%

### 5.2 Scalability

**Horizontal Scaling**
- API Service: 3-10 pods
- Workers: 10-30 pods (weighted queue depth)
- Min/max configured per component

**Growth Capacity**
- Current design supports 150M notifications/day (3x target)

### 5.3 Reliability and Availability

**Availability Targets**
- Service availability: 99.9%
- Component SLAs: APIM 99.95%, AKS 99.95%, Redis 99.9%, Cosmos DB 99.999%

**Fault Tolerance**
- Multi-AZ deployment
- Redis clustering
- Cosmos DB multi-region

**Data Durability**
- Service Bus: 3-zone replication
- At-least-once delivery guarantee

### 5.4 Observability

**Metrics**
- System: request rate, latency (P50/P95/P99), error rate
- Business: notifications by channel/priority, delivery success rate

**Logging**
- Structured JSON logs
- PII redaction
- Retention: 30 days hot, 90 days cold

**Distributed Tracing**
- W3C Trace Context
- Application Insights integration

**Dashboards**
- Operational: API metrics, queue depths, worker throughput
- Business: Daily volume, delivery rates, cost per notification

---

## 6. Implementation Plan

### 6.1 Phases

**Phase 1: Foundation** (Weeks 1-3)
- Infrastructure setup (AKS, Service Bus Standard, Cosmos DB, Redis, APIM, VNET)

**Phase 2: Core Services** (Weeks 4-8)
- API Service, Workers with Smart Router, Channel Handlers

**Phase 3: Data Management** (Weeks 9-11)
- Cosmos DB design, preference APIs, template management

**Phase 4: Testing & Hardening** (Weeks 12-14)
- Unit/integration/load tests, performance tuning

**Phase 5: Pilot & Migration** (Weeks 15-20)
- Pilot, gradual migration of all services

**Total**: 20 weeks (5 months) vs 26 weeks for event-driven

### 6.2 Dependencies

Same as Proposal 1 (Azure subscription, provider accounts, Azure AD, consuming services)

### 6.3 Risks and Mitigations

**Risk 1: Redis cache failure causes Cosmos DB overload**
- Mitigation: Request coalescing, circuit breaker to database, Redis clustering

**Risk 2: Worker state synchronization issues**
- Mitigation: Redis pub/sub for state sharing, conservative rate limits

**Risk 3: Queue head-of-line blocking**
- Mitigation: Priority-based pull, separate worker pools if needed

---

## 7. Operational Considerations

### 7.1 Deployment

**Strategy**: Blue-green with gradual traffic shift
**Automation**: Azure DevOps CI/CD
**Rollback**: Automated on health check failure

### 7.2 Operations and Maintenance

**Routine Operations**
- Daily: Monitor dashboards, review DLQ
- Weekly: Performance trends, cost reports
- Monthly: Secret rotation, DR drills

**Runbooks**
- High queue depth
- Low delivery success rate
- Redis unavailable
- Cosmos DB throttling

### 7.3 Cost Estimates

**Monthly Costs at 50M/day**

*Provider Costs:* $82,115/month

*Azure Infrastructure:*
- AKS: $600/month
- Service Bus Standard: $700/month (4 queues)
- Cosmos DB: $4,100/month (7,000 RU/s with caching)
- Redis Premium: $1,200/month (26 GB, multi-AZ)
- APIM Premium: $2,800/month
- Monitoring: $400/month
- **Subtotal: $9,800/month**

**Total: $91,915/month** ($1,200/month less than event-driven)

**Optimized with batching/routing: $68-72k/month**

**At 150M/day: ~$260k/month**

---

## 8. Appendices

### 8.1 Architecture Decision Records

See [ADRs - Proposal 2](./adrs-proposal-2.md)

### 8.2 References

Same as Proposal 1

### 8.3 Glossary

See `../../sources/glossary.md`

---

**End of Architecture Specification**
