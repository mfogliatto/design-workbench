# Architecture Specification: Customer Notification Service (Proposal 1 - Event-Driven)

**Version:** 1.0  
**Date:** November 21, 2025  
**Status:** Detailed Design  
**Author(s):** Architecture Team

---

## 1. Executive Summary

The Customer Notification Service (CNS) implements an event-driven architecture to centralize notification delivery across 15+ internal microservices. The system leverages Azure Service Bus topics with subscription-based routing to achieve high throughput (50M+ notifications/day), excellent scalability (3x growth capacity), and zero-message-loss guarantees. By decoupling notification submission from processing through events, the architecture enables independent scaling of components, isolated failure domains, and flexible extensibility for future channels.

**Key architectural highlights:**
- Event-driven communication via Azure Service Bus Topics eliminates tight coupling between API and processors
- Channel-specific processors (email, SMS, push) scale independently based on workload
- Multi-region Cosmos DB deployment ensures data residency compliance (GDPR)
- At-least-once delivery guarantees prevent message loss during failures
- Natural audit trail through event logging simplifies compliance and debugging

**Target metrics:**
- 99.9% service availability (43 minutes downtime/month)
- 580 msg/sec average throughput, 2,000 msg/sec peak capacity
- P95 API latency < 200ms (async), < 2s (sync)
- P95 end-to-end delivery < 30 seconds
- 99%+ delivery success rate for critical notifications

---

## 2. System Overview

### 2.1 Purpose and Scope

**Purpose**  
Centralize and standardize customer notification delivery across email, SMS, push notifications, and in-app messages. Replace scattered notification logic across 15+ services with a single, reliable, compliant notification platform.

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

1. **Reliability**: Zero message loss with at-least-once delivery guarantee; 99% delivery success rate
2. **Scalability**: Handle 50M notifications/day with capacity for 3x growth (150M/day)
3. **Performance**: P95 API latency < 200ms async, < 2s sync; critical notifications delivered in < 5 seconds
4. **Compliance**: GDPR data residency, CCPA opt-out management, CAN-SPAM unsubscribe handling
5. **Cost Efficiency**: Reduce notification costs by 30% through intelligent batching and routing
6. **Operational Excellence**: Comprehensive observability, zero-downtime deployments, clear runbooks

### 2.3 Success Criteria

- **Functional**: All 15 services migrated within 6 months; zero production incidents during migration
- **Performance**: 99.9% of API requests complete within SLA; no queue backlogs > 100k messages
- **Reliability**: < 1 critical incident per month; MTTR < 30 minutes
- **Cost**: Monthly operational costs < $75k after optimization (vs $70k current distributed costs)
- **Compliance**: Zero privacy violations; all opt-out requests honored within 24 hours

---

## 3. Architecture Overview

### 3.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Consuming Services Layer                           │
│          Order Service │ Account Service │ Promotion Service │ ...        │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │ REST API (HTTPS)
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      Azure API Management (APIM)                          │
│         OAuth 2.0 Auth │ Rate Limiting │ Request Validation               │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     CNS API Gateway (AKS - 5 pods)                        │
│  Request Validation │ Preference Lookup │ Event Publishing                │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │ NotificationSubmitted Event
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│          Azure Service Bus Topic: notifications-submitted                 │
│     Subscriptions: email-filter │ sms-filter │ push-filter                │
└─────────┬────────────────────┬────────────────────┬──────────────────────┘
          │                    │                    │
          ▼                    ▼                    ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Email Processor  │  │  SMS Processor   │  │  Push Processor  │
│   (AKS Pods)     │  │   (AKS Pods)     │  │   (AKS Pods)     │
│ SendGrid API     │  │   Twilio API     │  │  Azure NH API    │
└──────┬───────────┘  └──────┬───────────┘  └──────┬───────────┘
       │                     │                     │
       └─────────────────────┴─────────────────────┘
                             │ NotificationStatus Event
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│          Azure Service Bus Topic: notifications-status                    │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                  Status Tracker Service (AKS - 3 pods)                    │
│    Status Updates │ Webhook Triggers │ DLQ Management                     │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│               Azure Cosmos DB (Multi-Region: US West, EU West)            │
│  Customer Preferences │ Tracking Records │ Templates │ Audit Logs         │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Key Components

**Azure API Management (APIM)**
- Entry point for all consuming services
- Enforces OAuth 2.0 client credentials flow (Azure AD integration)
- Rate limiting: 100 req/sec per consuming service
- Request/response validation against OpenAPI schema
- Response compression and caching for status queries

**CNS API Gateway**
- Stateless microservice running on AKS (3-5 pods, HPA enabled)
- Validates notification payload (recipient, channel, template, data)
- Assigns unique tracking ID (ULID format for time-sortable IDs)
- Looks up customer preferences from Cosmos DB (with 5-min in-memory cache)
- Checks opt-out rules, quiet hours (based on customer timezone), and invalid contacts
- Publishes NotificationSubmitted event to Service Bus Topic
- For sync requests: subscribes to correlation-filtered status events and waits (timeout 2s)
- For async requests: returns tracking ID immediately (200 OK)

**Azure Service Bus Topics**
- **Topic: notifications-submitted** - Fan-out to channel-specific processors
  - Subscription: email-filter (SQL filter: `channel='email' OR channel='all'`)
  - Subscription: sms-filter (SQL filter: `channel='sms' OR channel='all'`)
  - Subscription: push-filter (SQL filter: `channel='push' OR channel='all'`)
  - Message properties: trackingId, priority, correlationId, timestamp
  - TTL: 24 hours (expired messages moved to DLQ)
- **Topic: notifications-status** - Status updates from processors to tracker
  - Subscription: status-updates (no filter, all messages)
  - Message properties: trackingId, status, attempt, timestamp, errorCode
  - TTL: 1 hour (status updates expire quickly)

**Channel Processors (Email, SMS, Push)**
- Specialized workers deployed as separate AKS deployments (independent scaling)
- Pull messages from respective Service Bus subscriptions (PeekLock mode)
- Render templates by merging with customer data (uses Handlebars.js engine)
- Implement rate limiting to respect provider constraints (token bucket algorithm)
- Call external provider APIs with retry logic (exponential backoff: 1s, 2s, 4s, 8s, 16s)
- Handle provider-specific errors (bounces, invalid numbers, unregistered devices)
- Publish NotificationStatus events for success/failure outcomes
- Complete or abandon Service Bus message based on result
- Auto-scale based on subscription queue depth (target: < 1000 messages)

**Status Tracker Service**
- Consumes status events from channel processors
- Updates notification tracking records in Cosmos DB (upsert operation)
- Triggers HTTP webhook callbacks to consuming services (if configured)
- Moves failed notifications to retry queue after exponential backoff delay
- Moves notifications to dead-letter queue after 5 failed attempts
- Emits correlation-filtered events for synchronous API requests
- Maintains metrics on delivery rates, attempt counts, and failure reasons

**Azure Cosmos DB**
- SQL API with multi-region write enabled (US West, EU West for data residency)
- Container: `customer-preferences` (partition key: `/customerId`, 10 GB, 3,000 RU/s)
- Container: `notification-tracking` (partition key: `/trackingId`, 500 GB, 5,000 RU/s)
- Container: `templates` (partition key: `/templateId`, 1 GB, 500 RU/s)
- Container: `audit-logs` (partition key: `/date`, 1 TB, 1,500 RU/s, TTL: 90 days)
- Consistency level: Session (balance between consistency and performance)
- Automatic indexing on frequently queried fields (customerId, trackingId, status, timestamp)

### 3.3 Key Technologies

**Azure Kubernetes Service (AKS)**
- Rationale: Industry-standard container orchestration; excellent scaling and deployment capabilities
- Configuration: 2 node pools (system and user), Standard_D4s_v3 VMs, auto-scaling enabled
- Benefits: Rolling updates, health checks, resource limits, horizontal pod autoscaling

**Azure Service Bus Premium**
- Rationale: Guaranteed throughput (8 MB/sec), predictable latency, advanced features (topics, filters, sessions)
- Configuration: 2 messaging units, geo-disaster recovery enabled, duplicate detection
- Benefits: At-least-once delivery, message durability, automatic dead-lettering

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

#### 4.1.1 API Gateway Component

**Responsibilities**
- Accept HTTP POST requests at `/api/v1/notifications` and `/api/v1/notifications/batch`
- Validate request schema (recipient, channel, template, data, priority, metadata)
- Authenticate and authorize requests via Azure AD OAuth 2.0 tokens
- Assign unique tracking ID (ULID format: timestamp-sortable, 26 characters)
- Look up customer preferences from Cosmos DB (cache-aside pattern, 5-min TTL)
- Apply business rules: opt-out checks, quiet hours enforcement, invalid contact filtering
- Publish NotificationSubmitted event to Service Bus Topic
- Handle synchronous vs asynchronous request patterns
- Return tracking ID or delivery status based on request type

**Interface Specification**

```
POST /api/v1/notifications
Authorization: Bearer {OAuth2-token}
Content-Type: application/json

Request Body:
{
  "recipient": {
    "customerId": "cust_123456789",
    "email": "customer@example.com",      // optional, if channel=email
    "phone": "+1234567890",               // optional, if channel=sms
    "deviceToken": "apns:device_token"    // optional, if channel=push
  },
  "channel": "email",                     // email | sms | push | in-app
  "template": {
    "id": "order-confirmation-v2",
    "version": "2.1.0"
  },
  "data": {
    "orderId": "ORD-2025-00123",
    "orderTotal": "$149.99",
    "deliveryDate": "2025-11-25"
  },
  "priority": "high",                     // critical | high | normal | low
  "metadata": {
    "serviceName": "order-service",
    "correlationId": "req-abc-123"
  },
  "options": {
    "async": true,                        // default: true (async mode)
    "webhookUrl": "https://service.com/webhook",
    "scheduledFor": null                  // ISO8601 timestamp for scheduled delivery
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

Response (Error - 400 Bad Request):
{
  "error": {
    "code": "INVALID_RECIPIENT",
    "message": "Customer has opted out of email notifications",
    "details": {
      "customerId": "cust_123456789",
      "optOutCategory": "marketing",
      "optOutDate": "2025-10-15T08:30:00Z"
    }
  }
}
```

**Scaling Strategy**
- Horizontal Pod Autoscaler (HPA): target CPU 70%, target memory 80%
- Min replicas: 3, Max replicas: 10
- Scale-up: add 1 pod per 200 req/sec sustained for 30 seconds
- Scale-down: remove 1 pod after 5 minutes below target

**Error Handling**
- Invalid request schema → 400 Bad Request
- Authentication failure → 401 Unauthorized
- Rate limit exceeded → 429 Too Many Requests (with Retry-After header)
- Customer opted out → 400 Bad Request with opt-out details
- Quiet hours violation (non-critical) → 202 Accepted with delayed delivery
- Service Bus unavailable → 503 Service Unavailable (circuit breaker)
- Cosmos DB unavailable → 503 Service Unavailable (serve from cache if possible)

#### 4.1.2 Channel Processor Components

**Email Processor**

*Responsibilities*
- Subscribe to email-filter subscription on notifications-submitted topic
- Retrieve and render email template from Cosmos DB
- Merge template with customer data using Handlebars.js
- Call SendGrid API with rendered content
- Handle SendGrid responses and errors (bounces, spam complaints, rate limits)
- Publish NotificationStatus event with outcome
- Complete or abandon Service Bus message

*Provider Integration*
```
SendGrid API Call:
POST https://api.sendgrid.com/v3/mail/send
Authorization: Bearer {sendgrid-api-key}
Content-Type: application/json

{
  "personalizations": [{
    "to": [{"email": "customer@example.com"}],
    "custom_args": {
      "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8"
    }
  }],
  "from": {"email": "noreply@company.com", "name": "Company Name"},
  "subject": "Your order has shipped!",
  "content": [{
    "type": "text/html",
    "value": "<html>...</html>"
  }]
}

Success Response (202 Accepted):
{
  "message_id": "msg_sg_xyz123"
}

Error Response (429 Rate Limit):
{
  "errors": [{"message": "Rate limit exceeded", "field": null}]
}
```

*Rate Limiting*
- Token bucket algorithm: 10,000 tokens/sec (SendGrid limit), refill rate 10,000/sec
- Safety margin: 90% of provider limit (9,000 effective emails/sec)
- Delay requests when bucket empty; publish back to topic with delay header

*Scaling Strategy*
- HPA based on Service Bus subscription queue depth
- Target: < 1,000 messages in queue
- Min replicas: 2, Max replicas: 15
- Scale metric: messages/pod should stay < 100

**SMS Processor**

*Responsibilities*
- Subscribe to sms-filter subscription
- Render SMS template (plain text, 160 char limit awareness)
- Call Twilio API with rendered content
- Handle Twilio responses (invalid numbers, carrier errors, undeliverable)
- Publish NotificationStatus event
- Complete or abandon message

*Provider Integration*
```
Twilio API Call:
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
  "status": "queued",
  "error_code": null
}

Error Response (400 Invalid Number):
{
  "code": 21211,
  "message": "The 'To' number is not a valid phone number."
}
```

*Rate Limiting*
- Token bucket: 1,000 tokens/sec (Twilio limit), safety margin 90% = 900/sec
- Twilio specific: respect per-destination rate limits (1 msg/sec per number)

*Scaling Strategy*
- Min replicas: 2, Max replicas: 10
- Target: < 500 messages in queue

**Push Processor**

*Responsibilities*
- Subscribe to push-filter subscription
- Render push notification payload (title, body, data)
- Call Azure Notification Hubs API
- Handle responses (unregistered devices, platform-specific errors)
- Publish NotificationStatus event

*Provider Integration*
```
Azure Notification Hubs API Call:
POST https://{namespace}.servicebus.windows.net/{hub}/messages?api-version=2015-01
Authorization: SharedAccessSignature {signature}
Content-Type: application/json
ServiceBusNotification-Format: apple (or gcm, fcm)

{
  "aps": {
    "alert": {
      "title": "Order Shipped",
      "body": "Your order ORD-2025-00123 has shipped!"
    },
    "badge": 1,
    "sound": "default"
  },
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8"
}

Success Response (201 Created):
{
  "notificationId": "nh_notif_xyz123"
}
```

*Rate Limiting*
- Token bucket: 10,000 tokens/sec (Azure NH limit), safety margin 90% = 9,000/sec

*Scaling Strategy*
- Min replicas: 2, Max replicas: 8
- Target: < 2,000 messages in queue (push notifications are fast)

#### 4.1.3 Status Tracker Component

**Responsibilities**
- Subscribe to notifications-status topic
- Parse status events and extract tracking metadata
- Update notification tracking records in Cosmos DB (upsert by trackingId)
- Trigger HTTP webhook callbacks to consuming services (fire-and-forget, 5s timeout)
- Implement retry logic for failed notifications:
  - 1st failure: retry after 1 second
  - 2nd failure: retry after 2 seconds
  - 3rd failure: retry after 4 seconds
  - 4th failure: retry after 8 seconds
  - 5th failure: retry after 16 seconds
  - After 5 failures: move to dead-letter queue
- Emit correlation-filtered status events for synchronous API requests
- Maintain delivery rate metrics and alert on anomalies

**Status Event Schema**
```json
{
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "status": "sent",              // queued | sent | delivered | failed | bounced
  "channel": "email",
  "provider": "sendgrid",
  "providerMessageId": "msg_sg_xyz123",
  "attempt": 1,
  "timestamp": "2025-11-21T10:15:32Z",
  "errorCode": null,
  "errorMessage": null,
  "correlationId": "req-abc-123"  // for sync requests
}
```

**Cosmos DB Tracking Record Schema**
```json
{
  "id": "01HZKM8XQFZ9B3G7H8J5K6M7N8",       // tracking ID
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
      "errorCode": null
    }
  ],
  "metadata": {
    "serviceName": "order-service",
    "correlationId": "req-abc-123"
  },
  "createdAt": "2025-11-21T10:15:30Z",
  "updatedAt": "2025-11-21T10:15:32Z",
  "ttl": 7776000                             // 90 days in seconds
}
```

**Webhook Callback**
```
POST https://service.company.com/webhook/notifications
Content-Type: application/json
X-CNS-Signature: {HMAC-SHA256 signature for verification}

{
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "status": "delivered",
  "timestamp": "2025-11-21T10:16:00Z",
  "channel": "email",
  "metadata": {
    "correlationId": "req-abc-123"
  }
}
```

**Scaling Strategy**
- Min replicas: 3, Max replicas: 8
- Scale based on status topic subscription queue depth
- Target: < 5,000 messages in queue

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

**Audit Log**
```json
{
  "id": "audit_01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "date": "2025-11-21",                      // partition key
  "timestamp": "2025-11-21T10:15:30Z",
  "eventType": "preference.updated",
  "customerId": "cust_123456789",
  "actor": "customer",
  "changes": {
    "field": "optOuts.categories",
    "oldValue": [],
    "newValue": ["marketing"]
  },
  "ipAddress": "203.0.113.42",
  "userAgent": "Mozilla/5.0...",
  "ttl": 7776000                              // 90 days
}
```

#### 4.2.2 Database Partitioning Strategy

**customer-preferences container**
- Partition key: `/customerId`
- Rationale: Preferences are always queried by customerId; provides perfect distribution
- Expected partitions: 12M customers → 12M logical partitions
- Hot partition mitigation: No VIP customers significantly hotter than others

**notification-tracking container**
- Partition key: `/trackingId`
- Rationale: Tracking queries use trackingId; each notification is independent
- Expected partitions: 50M/day × 90 days = 4.5B documents → 4.5B logical partitions
- Query patterns: Single-document reads by trackingId (efficient point reads)

**templates container**
- Partition key: `/templateId`
- Rationale: Templates queried by templateId; small dataset (~500 templates)
- Strategy: All templates cached in API Gateway and processors; database is source of truth

**audit-logs container**
- Partition key: `/date` (YYYY-MM-DD format)
- Rationale: Compliance queries typically date-range scoped; enables efficient TTL
- Expected partitions: 1 partition per day; old partitions automatically deleted after 90 days

#### 4.2.3 Data Flow Patterns

**Write Path (Notification Submission)**
1. API Gateway receives request → validates schema
2. API Gateway reads customer preferences from Cosmos DB (cache-aside, 5-min TTL)
3. API Gateway publishes event to Service Bus Topic (durable write)
4. API Gateway writes initial tracking record to Cosmos DB (async, fire-and-forget)
5. API Gateway returns response to caller

**Read Path (Status Query)**
1. Client sends GET /api/v1/notifications/{trackingId}
2. API Gateway queries Cosmos DB by trackingId (point read, ~10ms)
3. API Gateway returns tracking record with status and attempts

**Update Path (Status Update)**
1. Channel processor sends notification → receives provider response
2. Processor publishes status event to Service Bus Topic
3. Status Tracker consumes event → upserts tracking record in Cosmos DB
4. Status Tracker triggers webhook callback (if configured)

**Preference Update Path**
1. Consuming service sends PUT /api/v1/preferences/{customerId}
2. API Gateway validates preference update
3. API Gateway writes audit log entry to Cosmos DB
4. API Gateway updates preference record in Cosmos DB
5. API Gateway invalidates cache entry (cache-aside pattern)

### 4.3 APIs and Interfaces

#### 4.3.1 Public REST API

**Base URL**: `https://api.company.com/notifications/v1`

**Authentication**: OAuth 2.0 Client Credentials (Azure AD)
```
POST https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id={client-id}
&client_secret={client-secret}
&scope=https://api.company.com/.default
&grant_type=client_credentials

Response:
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Endpoints**

`POST /notifications` - Submit single notification
- Request: Notification object (see 4.1.1)
- Response: 202 Accepted (async) or 200 OK (sync)
- Rate limit: 100 req/sec per client

`POST /notifications/batch` - Submit batch notification
- Request: Array of notification objects (max 1000)
- Response: 202 Accepted with array of tracking IDs
- Rate limit: 10 req/sec per client

`GET /notifications/{trackingId}` - Query notification status
- Response: Tracking record with status and attempts
- Rate limit: 1000 req/sec per client
- Caching: 5-second cache at APIM layer

`GET /notifications?customerId={id}&status={status}&from={date}&to={date}` - Query notifications by criteria
- Response: Paginated list of tracking records (max 100 per page)
- Rate limit: 100 req/sec per client

`GET /preferences/{customerId}` - Get customer preferences
- Response: Preference object
- Rate limit: 1000 req/sec per client

`PUT /preferences/{customerId}` - Update customer preferences
- Request: Partial preference object
- Response: 200 OK with updated preferences
- Rate limit: 100 req/sec per client

**Error Response Format**
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "value"
    },
    "traceId": "00-abc123...",
    "timestamp": "2025-11-21T10:15:30Z"
  }
}
```

**Error Codes**
- `INVALID_REQUEST` (400): Malformed request body
- `AUTHENTICATION_FAILED` (401): Invalid or expired token
- `FORBIDDEN` (403): Client not authorized for operation
- `NOT_FOUND` (404): Resource not found (tracking ID, customer ID)
- `RATE_LIMIT_EXCEEDED` (429): Too many requests
- `INTERNAL_ERROR` (500): Unexpected server error
- `SERVICE_UNAVAILABLE` (503): Downstream dependency unavailable

#### 4.3.2 Internal Event Schemas

**NotificationSubmitted Event**
```json
{
  "eventType": "NotificationSubmitted",
  "eventId": "evt_01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "timestamp": "2025-11-21T10:15:30Z",
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "correlationId": "req-abc-123",
  "priority": "high",
  "channel": "email",
  "recipient": {
    "customerId": "cust_123456789",
    "email": "customer@example.com"
  },
  "template": {
    "id": "order-confirmation-v2",
    "version": "2.1.0"
  },
  "data": {
    "orderId": "ORD-2025-00123",
    "orderTotal": "$149.99"
  },
  "metadata": {
    "serviceName": "order-service"
  },
  "options": {
    "webhookUrl": "https://service.com/webhook"
  }
}
```

**NotificationStatus Event**
```json
{
  "eventType": "NotificationStatus",
  "eventId": "evt_01HZKM8YRZB3C4D5E6F7G8H9J0",
  "timestamp": "2025-11-21T10:15:32Z",
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "correlationId": "req-abc-123",
  "status": "sent",
  "channel": "email",
  "provider": "sendgrid",
  "providerMessageId": "msg_sg_xyz123",
  "attempt": 1,
  "latency": 120,
  "errorCode": null,
  "errorMessage": null
}
```

#### 4.3.3 Provider Webhooks

**SendGrid Webhook**
- Endpoint: `POST /webhook/sendgrid`
- Events: delivered, bounce, dropped, spam_report, unsubscribe
- Verification: SendGrid includes signature in headers; verify using shared secret

**Twilio Webhook**
- Endpoint: `POST /webhook/twilio`
- Events: delivered, failed, undelivered
- Verification: Twilio includes signature in X-Twilio-Signature header

**Azure Notification Hubs Feedback**
- Mechanism: Poll feedback API every 5 minutes
- Feedback types: Expired tokens, unregistered devices
- Action: Mark device tokens as invalid in customer preferences

### 4.4 Security Architecture

#### 4.4.1 Authentication and Authorization

**Service-to-Service Authentication**
- Protocol: OAuth 2.0 Client Credentials flow
- Provider: Azure AD (Entra ID)
- Token lifetime: 1 hour
- Token caching: API Gateway caches tokens and refreshes before expiry
- Scope: `https://api.company.com/notifications.send`

**API Authorization**
- Consuming services registered as Azure AD applications
- Each service assigned permissions (scopes) based on requirements
- API Gateway validates token signature and claims
- Rate limits enforced per client ID

**Internal Component Authentication**
- Managed identities for Azure resources (AKS pods → Cosmos DB, Service Bus)
- No credentials stored in application code or configuration
- Role-Based Access Control (RBAC) at Azure resource level

#### 4.4.2 Data Protection

**Encryption at Rest**
- Cosmos DB: Automatic encryption using Microsoft-managed keys
- Service Bus: Messages encrypted at rest using 256-bit AES
- Storage accounts: SSE (Storage Service Encryption) enabled
- Option: Customer-managed keys (CMK) via Azure Key Vault for additional control

**Encryption in Transit**
- TLS 1.3 for all HTTPS endpoints
- Service Bus connections use AMQP over TLS
- Cosmos DB connections use HTTPS only (TCP disabled)
- Certificate pinning for provider APIs (SendGrid, Twilio)

**PII Data Protection**
- Email addresses, phone numbers classified as PII
- Cosmos DB uses encryption at rest (email/phone encrypted in storage)
- Logging: PII redacted from application logs (masked as `***@***.com`)
- Audit logs: Store only hashed versions of contact info for compliance queries

#### 4.4.3 Secrets Management

**Azure Key Vault**
- All secrets stored in Azure Key Vault (provider API keys, OAuth client secrets)
- AKS pods access secrets via Managed Identity
- Secret rotation: Automated monthly rotation for provider API keys
- Audit: All secret access logged for compliance

**Secret References**
- SendGrid API key: `keyvault://cns-kv/sendgrid-api-key`
- Twilio credentials: `keyvault://cns-kv/twilio-account-sid`, `keyvault://cns-kv/twilio-auth-token`
- Azure Notification Hubs: Connection string in Key Vault

#### 4.4.4 Network Security

**VNET Integration**
- AKS cluster deployed in dedicated VNET (10.1.0.0/16)
- Cosmos DB private endpoint in VNET (no public internet access)
- Service Bus private endpoint in VNET
- API Management injected into VNET

**Network Segmentation**
- API Gateway subnet: 10.1.1.0/24
- Worker pods subnet: 10.1.2.0/23 (larger for auto-scaling)
- Data services subnet: 10.1.4.0/24 (Cosmos DB, Service Bus private endpoints)
- Network Security Groups (NSGs): Deny all except required ports

**Outbound Connectivity**
- Provider APIs (SendGrid, Twilio, Azure NH): Outbound via Azure NAT Gateway
- Static IP addresses whitelisted with providers
- Azure Firewall: Inspect and log all outbound traffic

#### 4.4.5 Security Monitoring

**Threat Detection**
- Azure Defender for Kubernetes: Monitor AKS for suspicious activity
- Azure Defender for Cosmos DB: Detect anomalous database access patterns
- Application Insights: Track authentication failures, rate limit violations

**Security Logging**
- All API requests logged with client ID, IP address, timestamp
- Failed authentication attempts trigger alerts after 10 failures in 5 minutes
- Cosmos DB audit logs: Track all preference changes for GDPR compliance
- Log retention: 90 days in hot storage, 7 years in archive for financial notifications

---

## 5. Non-Functional Requirements

### 5.1 Performance

**Throughput Targets**
- Average: 580 notifications/sec sustained
- Peak: 2,000 notifications/sec for 10 minutes
- Burst: 5,000 notifications/sec for 30 seconds (queue buffering)

**Latency Targets**
- API Gateway response (async): P50 < 50ms, P95 < 200ms, P99 < 500ms
- API Gateway response (sync): P50 < 1s, P95 < 2s, P99 < 5s
- End-to-end delivery (request to provider send): P50 < 10s, P95 < 30s, P99 < 60s
- Critical notifications: P95 < 5s end-to-end

**Database Performance**
- Cosmos DB point reads: P50 < 5ms, P95 < 10ms
- Cosmos DB writes: P50 < 10ms, P95 < 20ms
- Preference cache hit rate: > 80%

**Provider Latency**
- SendGrid API: P50 < 200ms, P95 < 500ms
- Twilio API: P50 < 150ms, P95 < 400ms
- Azure NH API: P50 < 100ms, P95 < 300ms

### 5.2 Scalability

**Horizontal Scaling**
- API Gateway: 3-10 pods, scales at 70% CPU
- Email Processor: 2-15 pods, scales at queue depth > 1000
- SMS Processor: 2-10 pods, scales at queue depth > 500
- Push Processor: 2-8 pods, scales at queue depth > 2000
- Status Tracker: 3-8 pods, scales at queue depth > 5000

**Vertical Scaling**
- Cosmos DB: 10,000 RU/s provisioned (can burst to 100,000 RU/s with autoscale)
- Service Bus: 2 messaging units (can scale to 8 messaging units)

**Growth Capacity**
- Current design supports 150M notifications/day (3x target)
- Beyond 3x: Add more Service Bus messaging units, scale Cosmos DB to 30,000 RU/s
- Cost projection at 150M/day: ~$260k/month

### 5.3 Reliability and Availability

**Availability Targets**
- Service availability: 99.9% (43 minutes downtime/month)
- Individual component SLAs:
  - API Management: 99.95%
  - AKS: 99.95% (with availability zones)
  - Service Bus Premium: 99.9%
  - Cosmos DB: 99.999% (multi-region)

**Fault Tolerance**
- Multi-AZ deployment for all components
- Service Bus: Geo-disaster recovery enabled (paired region)
- Cosmos DB: Multi-region with automatic failover
- AKS: Pod anti-affinity rules (spread across nodes and zones)

**Data Durability**
- Service Bus: Messages replicated across 3 zones
- Cosmos DB: Multi-region replication (RPO < 5 minutes)
- At-least-once delivery guarantee (no message loss)

**Failure Recovery**
- Pod failure: Kubernetes restarts pod within 30 seconds
- Node failure: Pods rescheduled to healthy nodes within 2 minutes
- Zone failure: Traffic shifts to other zones (no impact)
- Region failure: Cosmos DB fails over automatically; Service Bus manual failover in < 5 minutes

**Circuit Breaker Configuration**
- Provider API failures: Open circuit after 10 consecutive failures, 30s timeout, retry with exponential backoff
- Cosmos DB failures: Open circuit after 5 consecutive 503 errors, serve from cache, 10s timeout

### 5.4 Observability

**Metrics**

*System Metrics* (collected every 30 seconds)
- API Gateway: request rate, response time (P50, P95, P99), error rate, active connections
- Channel Processors: processing rate, provider latency, retry rate, dead-letter count
- Service Bus: queue depth, message throughput, delivery latency
- Cosmos DB: RU consumption, request rate, latency, throttling events

*Business Metrics* (collected every 1 minute)
- Notifications submitted by channel/priority
- Delivery success rate by channel/provider
- Delivery latency by channel (submission to provider send)
- Opt-out rate by category
- Webhook callback success rate

**Logging**

*Structured Logging Format* (JSON)
```json
{
  "timestamp": "2025-11-21T10:15:30.123Z",
  "level": "INFO",
  "service": "api-gateway",
  "traceId": "00-abc123...",
  "spanId": "def456...",
  "message": "Notification queued successfully",
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "customerId": "cust_***6789",
  "channel": "email",
  "priority": "high",
  "durationMs": 45
}
```

*Log Levels*
- DEBUG: Detailed diagnostic info (disabled in production)
- INFO: Normal operational events (e.g., notification queued)
- WARN: Recoverable errors (e.g., provider rate limit, retrying)
- ERROR: Unrecoverable errors requiring investigation (e.g., message abandoned to DLQ)

*Log Retention*
- Hot storage (Log Analytics): 30 days
- Cold storage (Azure Blob): 90 days for operational logs, 7 years for audit logs

**Distributed Tracing**
- W3C Trace Context standard (traceId, spanId propagation)
- Application Insights: End-to-end traces across API Gateway → Service Bus → Processor → Provider
- Trace sampling: 100% for errors, 10% for success at high volume
- Trace retention: 90 days

**Dashboards**

*Operational Dashboard*
- API request rate, error rate, latency (P50, P95, P99)
- Service Bus queue depths by priority
- Channel processor throughput and provider latency
- Cosmos DB RU consumption and throttling

*Business Dashboard*
- Daily notification volume by channel
- Delivery success rate by channel and priority
- Top failure reasons (invalid contacts, opt-outs, provider errors)
- Cost per notification by channel

**Alerting**

*Critical Alerts* (Page on-call engineer)
- API error rate > 5% for 5 minutes
- Queue depth > 100k messages for 10 minutes
- Delivery success rate < 90% for 10 minutes
- Service Bus or Cosmos DB unavailable

*Warning Alerts* (Slack notification)
- API latency P95 > 300ms for 10 minutes
- Provider latency P95 > 1s for 5 minutes
- Dead-letter queue count > 1000 messages
- Cosmos DB RU consumption > 90% for 5 minutes

*Info Alerts* (Dashboard notification)
- Unusual traffic pattern detected (10x normal)
- Provider rate limit warnings
- Cache hit rate < 70%

---

## 6. Implementation Plan

### 6.1 Phases

**Phase 1: Foundation (Weeks 1-4)**
- Set up Azure infrastructure (AKS, Service Bus, Cosmos DB, APIM)
- Configure VNET, NSGs, private endpoints
- Deploy Kubernetes base manifests (namespaces, service accounts, RBAC)
- Set up CI/CD pipelines (Azure DevOps or GitHub Actions)
- Configure monitoring (Application Insights, Log Analytics, dashboards)
- Deliverable: Infrastructure ready for application deployment

**Phase 2: Core Services (Weeks 5-10)**
- Implement API Gateway service
  - Request validation, authentication, preference lookup
  - Event publishing to Service Bus
  - Async and sync response patterns
- Implement Channel Processors (email, SMS, push)
  - Service Bus subscription consumers
  - Template rendering
  - Provider integrations (SendGrid, Twilio, Azure NH)
  - Retry logic and error handling
- Implement Status Tracker service
  - Status event consumer
  - Cosmos DB tracking record updates
  - Webhook callbacks
- Deliverable: End-to-end notification flow working in dev environment

**Phase 3: Data Management (Weeks 11-13)**
- Design and create Cosmos DB containers (preferences, tracking, templates, audit logs)
- Implement preference management APIs (GET, PUT)
- Build template management functionality
- Create data migration scripts for existing preferences
- Implement audit logging for compliance
- Deliverable: Complete data layer with migration scripts

**Phase 4: Testing & Hardening (Weeks 14-17)**
- Unit tests (80% code coverage target)
- Integration tests (end-to-end flows)
- Load testing (simulate 2,000 msg/sec peak, validate auto-scaling)
- Chaos engineering (inject failures, validate recovery)
- Security testing (penetration testing, vulnerability scanning)
- Performance tuning (optimize queries, caching, scaling policies)
- Deliverable: Production-ready service with passing tests

**Phase 5: Pilot & Migration (Weeks 18-26)**
- Deploy to production with feature flags (traffic disabled)
- Pilot with 1-2 low-volume services (~1M/day) - Weeks 18-20
- Validate metrics, logs, alerts; tune configuration
- Migrate 5 medium-volume services (~20M/day) - Weeks 21-24
- Monitor performance and cost; adjust scaling policies
- Migrate remaining 8 services - Weeks 25-26
- Deliverable: All services migrated, old notification code decommissioned

### 6.2 Dependencies

**External Dependencies**
- Azure subscription with required quotas (AKS nodes, Cosmos DB throughput, Service Bus messaging units)
- SendGrid, Twilio, Azure Notification Hubs accounts with sufficient capacity
- Azure AD tenant for OAuth 2.0 service-to-service authentication
- Network team: VNET setup, NSG rules, private endpoint approvals

**Internal Dependencies**
- Consuming services: Adopt wrapper library or update to call CNS API
- Template owners: Migrate templates to new format, validate rendering
- Compliance team: Review GDPR/CCPA implementation, approve data residency strategy
- Security team: Review architecture, approve penetration testing, validate secrets management

**Team Dependencies**
- Platform team: AKS cluster setup, monitoring configuration
- Data team: Cosmos DB optimization, query tuning
- SRE team: Runbooks, on-call training, incident response procedures

### 6.3 Risks and Mitigations

**Risk 1: Provider rate limits overwhelm system during migration**
- Impact: HIGH - Could cause notification delays or failures
- Probability: MEDIUM - Many services migrating simultaneously
- Mitigation: Gradual migration with traffic throttling; implement aggressive rate limiting with 10% safety margin; provider contract negotiation for temporary rate limit increase

**Risk 2: Event-driven complexity leads to debugging difficulties**
- Impact: MEDIUM - Slower incident resolution
- Probability: HIGH - Common with event-driven architectures
- Mitigation: Comprehensive distributed tracing with correlation IDs; detailed runbooks for common failure scenarios; invest in observability tooling upfront

**Risk 3: Cosmos DB hot partitions degrade performance**
- Impact: HIGH - Could violate latency SLAs
- Probability: LOW - Partition keys designed to distribute evenly
- Mitigation: Monitor partition metrics; implement caching for preference reads; use Cosmos DB autoscale to absorb bursts

**Risk 4: Service Bus queue backlogs during peak traffic**
- Impact: MEDIUM - Increased delivery latency
- Probability: MEDIUM - Peak traffic can exceed processing capacity
- Mitigation: Auto-scaling tuned aggressively (target low queue depth); alerting on queue depth; backpressure mechanism to slow down API if queue exceeds 100k

**Risk 5: Multi-region failover disrupts service**
- Impact: HIGH - Potential message loss or duplicate sends
- Probability: LOW - Rare region failures
- Mitigation: Automated failover testing (chaos engineering); idempotency keys in messages to prevent duplicates; Service Bus geo-disaster recovery configured

**Risk 6: Security vulnerability in dependency or code**
- Impact: CRITICAL - Data breach or compliance violation
- Probability: MEDIUM - Common in complex systems
- Mitigation: Automated dependency scanning (Dependabot, Snyk); regular penetration testing; security code reviews; PII data encryption and access controls

**Risk 7: Cost overruns due to underestimated usage**
- Impact: HIGH - Budget exceeded
- Probability: MEDIUM - Estimates based on projections
- Mitigation: Detailed cost monitoring with alerts; cost optimization iterations (batching, caching); monthly cost reviews; reserved capacity for predictable workloads

---

## 7. Operational Considerations

### 7.1 Deployment

**Deployment Strategy**
- Blue-green deployments for zero downtime
- Kubernetes rolling updates with health checks
- Deployment windows: Tuesday/Thursday 10am-12pm PT (low traffic period)
- Rollback plan: Automated rollback on health check failure (5 consecutive failures)

**Deployment Process**
1. Deploy new version to "green" environment (separate namespace or cluster)
2. Run smoke tests against green environment
3. Gradually shift traffic from blue to green (10%, 25%, 50%, 100% over 30 minutes)
4. Monitor metrics and error rates during shift
5. If errors spike: immediate rollback to blue
6. If successful: decommission blue after 24-hour soak period

**Canary Releases**
- For high-risk changes: Deploy to 5% of traffic for 24 hours
- Monitor metrics: error rate, latency, delivery success rate
- If stable: Increase to 50%, then 100% over 3 days
- Feature flags: Enable new features gradually per consuming service

**Deployment Automation**
- CI/CD pipeline: Azure DevOps (or GitHub Actions)
- Automated build, test, and deploy on merge to main branch
- Infrastructure as Code: Bicep templates for Azure resources, Helm charts for Kubernetes
- Manual approval gate for production deployments

### 7.2 Operations and Maintenance

**Routine Operations**

*Daily Tasks*
- Monitor dashboards for anomalies (queue depths, error rates, latency)
- Review dead-letter queue (investigate failed messages)
- Check provider rate limit consumption (ensure headroom)
- Review security alerts (failed auth attempts, unusual access patterns)

*Weekly Tasks*
- Review performance trends (identify degradation)
- Analyze cost reports (optimize usage)
- Review and triage alerts (tune thresholds)
- Check dependency updates (security patches)

*Monthly Tasks*
- Rotate secrets (provider API keys, OAuth client secrets)
- Review capacity planning (forecast growth)
- Disaster recovery drill (test failover procedures)
- Security review (vulnerability scan results, penetration test findings)

**Runbook Examples**

*Runbook: High Queue Depth Alert*
1. Check current queue depth across all priority levels
2. Verify worker pod health and scaling status (kubectl get pods, check HPA)
3. Check provider API health (recent error rates, latency)
4. If provider degraded: Circuit breaker should be open; wait for recovery
5. If workers at max scale: Temporarily increase max replicas
6. If neither: Check for message processing errors in logs (potential bug)
7. Escalate to engineering if unable to resolve in 15 minutes

*Runbook: Low Delivery Success Rate*
1. Identify which channel has low success rate (email, SMS, push)
2. Check provider status page for known issues
3. Review recent error messages from provider (bounces, invalid contacts, rate limits)
4. If provider issue: Enable circuit breaker manually, notify stakeholders
5. If contact data quality issue: Notify consuming services to validate contacts
6. If template issue: Identify problematic template, disable if necessary
7. Document findings and update monitoring thresholds

*Runbook: Cosmos DB Throttling*
1. Check current RU consumption and provisioned throughput
2. Identify hot partition (query metrics by partition key)
3. If hot partition: Investigate why (VIP customer, spike in preference updates)
4. Short-term: Increase provisioned RU/s to 15,000
5. Medium-term: Optimize queries (add indexes, reduce query scope)
6. Long-term: Rearchitect partition strategy if persistent hot partition

*Runbook: Service Bus Connection Failures*
1. Verify Service Bus health (Azure portal, service health dashboard)
2. Check AKS pod logs for connection error details
3. Verify managed identity permissions (RBAC roles)
4. If regional issue: Trigger manual geo-failover to paired region
5. If credentials issue: Rotate Service Bus connection strings
6. Monitor message loss (should be zero due to client-side retries)

**Maintenance Windows**
- Monthly patching: 1st Sunday of month, 2am-6am PT
- AKS cluster upgrades: Quarterly, rolling node pool upgrades
- Cosmos DB maintenance: Fully managed by Azure (no downtime)
- Service Bus maintenance: Fully managed by Azure (no downtime)

**Incident Response**

*Severity Levels*
- SEV1 (Critical): Service down or data loss; page on-call immediately
- SEV2 (High): Degraded performance or partial outage; notify on-call within 15 minutes
- SEV3 (Medium): Minor issue with workaround; notify during business hours
- SEV4 (Low): Cosmetic or documentation issue; backlog for sprint planning

*Incident Process*
1. Acknowledge alert and update status page
2. Triage: Identify affected components and impact
3. Mitigate: Apply immediate fix or workaround (rollback, scale, failover)
4. Communicate: Update stakeholders every 30 minutes until resolved
5. Resolve: Restore service to normal operation
6. Post-mortem: Within 5 business days, document root cause and action items

### 7.3 Cost Estimates

**Monthly Costs at 50M/day**

*Provider Costs*
- SendGrid: $3,300/month (33M emails @ $0.0001)
- Twilio: $78,750/month (10.5M SMS @ $0.0075)
- Azure Notification Hubs: $65/month (6.5M push @ $0.00001)
- **Subtotal: $82,115/month**

*Azure Infrastructure Costs*
- AKS: $600/month (Standard_D4s_v3 nodes, 3-5 nodes average)
- Azure Service Bus Premium: $1,340/month (2 messaging units)
- Azure Cosmos DB: $5,840/month (10,000 RU/s provisioned, multi-region, 500 GB storage)
- Azure Blob Storage: $50/month (1 TB hot tier for templates and logs)
- Azure API Management Premium: $2,800/month
- Azure Monitor / Application Insights: $400/month
- **Subtotal: $11,030/month**

**Total: $93,145/month**

*Compared to current state:* +$22,595/month (+32%)

**Cost Optimization Opportunities**
- Intelligent email batching: Save 15-20% on SendGrid costs (~$600/month)
- Smart channel selection (email-first): Save 20-30% on Twilio costs (~$18,000/month)
- Cosmos DB autoscale: Save 30% during low-traffic periods (~$1,750/month)
- Reserved capacity for AKS and Cosmos DB: Save 30-40% on compute (~$2,000/month)
- **Potential savings: $22,000-25,000/month**
- **Optimized cost: $68,000-70,000/month (near cost-neutral vs current $70k)**

**Cost at 150M/day (3x growth)**
- Provider costs: ~$246,000/month (linear scaling)
- Infrastructure: ~$18,000/month (Cosmos DB and Service Bus scale significantly; AKS grows modestly)
- **Total: ~$264,000/month ($1.76 per 1,000 notifications)**

---

## 8. Appendices

### 8.1 Architecture Decision Records (ADRs)

See separate consolidated ADR document: [ADRs for Proposal 1](./adrs-proposal-1.md)

### 8.2 References

**External Documentation**
- [Azure Service Bus Documentation](https://learn.microsoft.com/en-us/azure/service-bus-messaging/)
- [Azure Cosmos DB Best Practices](https://learn.microsoft.com/en-us/azure/cosmos-db/best-practice-guide)
- [SendGrid API Reference](https://docs.sendgrid.com/api-reference)
- [Twilio SMS API Reference](https://www.twilio.com/docs/sms/api)
- [Azure Notification Hubs Documentation](https://learn.microsoft.com/en-us/azure/notification-hubs/)

**Internal Documentation**
- Source files: `../../sources/` (overview, requirements, facts, questions-to-answer, glossary)
- High-level proposal: `../proposal-1-event-driven-architecture.md`
- Comparison with other proposals: `../README.md`

**Standards and Compliance**
- [GDPR Compliance Guide](https://gdpr.eu/)
- [CCPA Overview](https://oag.ca.gov/privacy/ccpa)
- [CAN-SPAM Act Requirements](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [OAuth 2.0 Client Credentials Grant](https://oauth.net/2/grant-types/client-credentials/)

### 8.3 Glossary

See comprehensive glossary in source file: `../../sources/glossary.md`

**Architecture-Specific Terms**

**Event-Driven Architecture**
Design pattern where components communicate primarily through events rather than direct API calls. Provides loose coupling and independent scalability.

**Topic Subscription Filter**
Azure Service Bus feature that routes messages to specific subscriptions based on message properties (e.g., channel=email routes only email notifications).

**At-Least-Once Delivery**
Guarantee that every message will be delivered at least one time, but may be delivered multiple times in failure scenarios. Requires idempotent message processing.

**PeekLock Mode**
Service Bus message retrieval mode where messages are locked (invisible to other consumers) but not deleted until explicitly completed. Enables retry on failure.

**Dead-Letter Queue (DLQ)**
Storage for messages that cannot be processed successfully after multiple attempts. Requires manual investigation and resolution.

**Correlation ID**
Unique identifier that links related messages and API requests together for distributed tracing and debugging.

**Horizontal Pod Autoscaler (HPA)**
Kubernetes feature that automatically scales pod count based on observed metrics (CPU, memory, or custom metrics like queue depth).

**Blue-Green Deployment**
Deployment strategy with two identical environments (blue and green). Traffic switches from blue to green after validation, enabling instant rollback.

---

**End of Architecture Specification**
