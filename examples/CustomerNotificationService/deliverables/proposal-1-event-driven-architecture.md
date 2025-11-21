# Proposal 1: Event-Driven Architecture

## 1. Executive Summary

### Problem Statement
The Customer Notification Service must handle 50M+ notifications daily across multiple channels while ensuring reliability, scalability, and compliance. The key challenges include managing high throughput (2,000 msg/sec peak), coordinating multiple external providers with different rate limits, tracking delivery status across distributed components, and supporting zero-message-loss guarantees during failures.

### Proposed Solution
This proposal leverages an **event-driven architecture** using Azure Service Bus as the central nervous system. The design decouples notification submission from processing through topic-based routing, enabling independent scaling of API ingress, channel processors, and provider integrations. Each notification flows through a series of events (submitted → validated → routed → sent → tracked), with each stage handled by specialized microservices that react to events and emit new ones. This creates a resilient, loosely-coupled system where components can fail, retry, and scale independently without blocking upstream services.

The architecture uses Azure Service Bus Topics with multiple subscriptions to fan-out notifications to channel-specific processors (email, SMS, push). Each processor manages its own retry logic, rate limiting, and provider integration. Status updates flow back through events to update Cosmos DB tracking records, enabling real-time status queries and webhook callbacks. The event-driven nature naturally supports both synchronous (wait for acknowledgment) and asynchronous (fire-and-forget) API patterns.

### Key Benefits
- **Natural decoupling**: Services communicate via events, eliminating tight coupling and synchronous dependencies between components
- **Independent scalability**: Each channel processor scales independently based on its workload (email vs SMS vs push)
- **Built-in resilience**: Message durability in Service Bus ensures zero message loss; failed components automatically resume from their last checkpoint
- **Flexible priority handling**: Topic filters route critical notifications to high-priority processors while normal traffic follows standard paths
- **Audit trail by default**: Every event creates a natural audit log, simplifying compliance and debugging

## 2. High-Level Architecture

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Consuming Services                                  │
│                 (Order Service, Account Service, etc.)                       │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ REST API
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Azure API Management (APIM)                             │
│              Authentication, Rate Limiting, Request Validation               │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CNS API Gateway (AKS)                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ • Validate request payload                                           │   │
│  │ • Assign tracking ID                                                 │   │
│  │ • Check customer preferences (cache-first from Cosmos DB)            │   │
│  │ • Publish NotificationSubmitted event to Service Bus Topic           │   │
│  │ • Return tracking ID (async) OR wait for sent confirmation (sync)   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ Event: NotificationSubmitted
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              Azure Service Bus Topic: notifications-submitted                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  Subscription:  │  │  Subscription:  │  │  Subscription:  │             │
│  │  email-filter   │  │  sms-filter     │  │  push-filter    │             │
│  │  (channel=email)│  │  (channel=sms)  │  │  (channel=push) │             │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘             │
└───────────┼────────────────────┼────────────────────┼─────────────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│  Email Processor    │ │   SMS Processor     │ │   Push Processor    │
│      (AKS Pods)     │ │     (AKS Pods)      │ │     (AKS Pods)      │
├─────────────────────┤ ├─────────────────────┤ ├─────────────────────┤
│ • Render template   │ │ • Render template   │ │ • Render template   │
│ • Rate limit        │ │ • Rate limit        │ │ • Rate limit        │
│ • Call SendGrid     │ │ • Call Twilio       │ │ • Call Azure NH     │
│ • Retry on failure  │ │ • Retry on failure  │ │ • Retry on failure  │
│ • Emit sent event   │ │ • Emit sent event   │ │ • Emit sent event   │
└──────────┬──────────┘ └──────────┬──────────┘ └──────────┬──────────┘
           │                       │                       │
           └───────────────────────┴───────────────────────┘
                                   │ Event: NotificationSent
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              Azure Service Bus Topic: notifications-status                   │
│                     (DeliveryStatus, ProviderResponse)                       │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Status Tracker Service (AKS)                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ • Update tracking record in Cosmos DB                                │   │
│  │ • Trigger webhooks for status changes                                │   │
│  │ • Move failed notifications to retry queue or DLQ                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Azure Cosmos DB (Multi-Region)                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ • Customer preferences (channel, opt-outs, quiet hours)              │   │
│  │ • Notification tracking (status, attempts, timestamps)               │   │
│  │ • Templates (with versioning)                                        │   │
│  │ • Audit logs (90-day retention)                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                      External Notification Providers                         │
│        SendGrid (Email)  │  Twilio (SMS)  │  Azure Notification Hubs        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Overview

**Azure API Management (APIM)**
- Single entry point for all consuming services
- Handles OAuth 2.0 authentication and authorization
- Enforces rate limits per consuming service
- Provides request validation and transformation
- Routes requests to CNS API Gateway pods

**CNS API Gateway**
- Lightweight stateless service running on AKS
- Validates notification requests and assigns tracking IDs
- Performs cache-first lookup of customer preferences from Cosmos DB
- Checks opt-out status and quiet hours before accepting notification
- Publishes `NotificationSubmitted` event to Service Bus
- For synchronous API calls: waits for `NotificationSent` event before responding
- For asynchronous API calls: returns tracking ID immediately

**Azure Service Bus Topics**
- `notifications-submitted`: Fan-out topic with channel-specific filters (email, SMS, push)
- `notifications-status`: Status update events from processors to tracker
- Provides message durability (at-least-once delivery guarantee)
- Supports message TTL and dead-letter queues
- Enables priority routing through message properties

**Channel Processors (Email, SMS, Push)**
- Specialized workers that subscribe to channel-specific topic filters
- Pull messages from Service Bus subscriptions
- Render templates with customer data
- Implement rate limiting to respect provider limits
- Call external providers (SendGrid, Twilio, Azure NH)
- Handle retries with exponential backoff
- Emit status events for success/failure
- Scale independently based on queue depth

**Status Tracker Service**
- Consumes status events from channel processors
- Updates notification tracking records in Cosmos DB
- Triggers webhook callbacks to consuming services
- Moves failed notifications to retry queue or DLQ after max attempts
- Provides correlation between requests and delivery outcomes

**Azure Cosmos DB**
- Multi-region deployment (US, EU) for data residency compliance
- Partitioned by `customerId` for efficient preference lookups
- Stores customer preferences with TTL-based caching
- Tracks notification delivery status and history
- Maintains template library with versioning
- 90-day retention for audit logs with automatic archival

### Data Flow

**Normal Flow (Asynchronous)**
1. Consuming service sends notification request to APIM
2. APIM validates authentication and forwards to API Gateway
3. API Gateway checks preferences, assigns tracking ID, publishes event
4. API Gateway returns tracking ID to caller (200 OK)
5. Service Bus routes event to appropriate channel processor(s)
6. Channel processor renders template and calls provider
7. Processor emits status event (sent/failed)
8. Status Tracker updates Cosmos DB and triggers webhooks

**Synchronous Flow**
1-3. Same as asynchronous flow
4. API Gateway subscribes to correlation-filtered status events
5-7. Same as asynchronous flow
8. Status Tracker emits status event with correlation ID
9. API Gateway receives status event and responds to caller

### Integration Points

- **External Providers**: SendGrid (email), Twilio (SMS), Azure Notification Hubs (push)
- **Internal Services**: 15+ consuming services via REST API through APIM
- **Azure AD**: OAuth 2.0 service-to-service authentication
- **Application Insights**: Distributed tracing and telemetry
- **Azure Monitor**: Metrics, alerting, and dashboards

## 3. Core Trade-offs

### Key Advantages
1. **Excellent scalability**: Each channel processor scales independently; can easily handle 3x growth by adding more pods
2. **Strong reliability**: Service Bus message durability eliminates message loss; failed processors resume from checkpoint
3. **Natural audit trail**: Every event creates a record; debugging and compliance are straightforward
4. **Flexible extensibility**: Adding new channels (in-app, WhatsApp) only requires new processor subscriptions
5. **Loose coupling**: Components can be deployed, updated, and scaled independently without coordination

### Key Disadvantages
1. **Increased complexity**: Event-driven systems are harder to reason about; debugging requires tracing through multiple components
2. **Eventual consistency**: Status updates lag behind actual sends; synchronous API pattern adds latency waiting for events
3. **Higher infrastructure costs**: Service Bus Premium ($1,340/month) required for throughput; multiple topic subscriptions increase message volume
4. **Testing challenges**: End-to-end testing requires event simulation; local development needs Service Bus emulator
5. **Operational overhead**: More moving parts to monitor; event ordering and duplicate handling add complexity

### Risk Summary
- **Event storms**: Retry logic could create cascading events; mitigate with exponential backoff and circuit breakers
- **Message poisoning**: Malformed events block subscriptions; mitigate with validation and automatic DLQ after N attempts
- **Cross-component debugging**: Distributed traces span many services; mitigate with comprehensive correlation IDs and Application Insights
- **Provider rate limits**: Bursts overwhelm providers despite rate limiting; mitigate with backpressure and queue depth monitoring

## 4. Feasibility Assessment

### Implementation Complexity: **Medium-High**

**Justification**: Event-driven architectures require careful design of event schemas, message handling, and error recovery. The team needs expertise in Service Bus topics/subscriptions, idempotency patterns, and distributed tracing. However, Azure provides mature services (Service Bus, Cosmos DB, AKS) that handle much of the heavy lifting. The main complexity is in coordinating multiple asynchronous components and ensuring end-to-end visibility.

**Development effort**:
- API Gateway: 3-4 weeks
- Channel processors: 2-3 weeks each (can parallelize)
- Status tracker: 2 weeks
- Infrastructure setup: 2 weeks
- Testing and debugging: 4 weeks
- **Total: ~4-5 months with 3-4 engineers**

### Operational Impact

**Changes required**:
- Operations team must learn Service Bus monitoring (queue depths, dead-letter queues, message throughput)
- Debugging shifts from request/response logs to distributed tracing with correlation IDs
- Incident response requires understanding event flows across multiple services
- New runbooks for handling stuck queues, poison messages, and event replays

**Benefits**:
- Each component can be deployed independently (blue-green deployments easier)
- Auto-scaling based on queue depth is straightforward
- Component failures isolated; email processor failure doesn't affect SMS

### Migration Approach

**Strategy: Parallel running with gradual cutover**

1. **Phase 1** (Month 1-2): Deploy CNS infrastructure and API in read-only mode; consuming services don't send real traffic yet
2. **Phase 2** (Month 3): Pilot with 1-2 low-volume services (~1M notifications/day); validate end-to-end flow and monitoring
3. **Phase 3** (Month 4-5): Migrate 5 medium-volume services (~20M/day total); tune scaling and rate limiting
4. **Phase 4** (Month 6): Migrate remaining services; decommission old notification code
5. **Rollback plan**: Feature flags in consuming services to instantly revert to old notification paths

**Zero-downtime approach**: Consuming services update configuration to call CNS API instead of provider APIs directly; no code changes required if wrapper libraries are used.

### Rough Order of Magnitude

**Development costs**:
- Engineering: 4 engineers × 5 months × $15k/month = $300k
- Infrastructure (pre-production): $5k/month × 5 months = $25k
- **Total development: ~$325k**

**Monthly operational costs** (at 50M/day volume):
- Provider costs: $82,115/month (SendGrid, Twilio, Azure NH)
- Azure infrastructure: $11,030/month (AKS, Service Bus, Cosmos DB, APIM)
- **Total: $93,145/month**

**Cost optimization potential**: With intelligent batching and channel routing, projected savings of $18-25k/month bring costs to ~$70k/month (near cost-neutral vs current $70k/month distributed costs).

**At 150M/day (3x growth)**:
- Provider costs: ~$246k/month (linear scaling)
- Infrastructure: ~$18k/month (Service Bus and Cosmos DB scale; AKS grows modestly)
- **Total: ~$264k/month** ($1.76 per 1000 notifications)

---

## 5. Detailed Design Documentation

This high-level proposal has been refined into comprehensive, implementation-ready documentation:

### 5.1 Architecture Specification
**Document**: [Architecture Specification - Proposal 1](./supporting-documents/architecture-spec-proposal-1.md)

The complete architecture specification includes:
- **Detailed Component Design**: Full specifications for API Gateway, Channel Processors (email, SMS, push), Status Tracker, with interface definitions, scaling strategies, and error handling
- **API Contracts**: Complete REST API documentation with request/response schemas, authentication, rate limiting, and error codes
- **Data Architecture**: Cosmos DB container designs, partitioning strategies, data models for preferences/tracking/templates/audit logs
- **Security Architecture**: Authentication flows, encryption (at rest and in transit), secrets management, network security, threat detection
- **Non-Functional Requirements**: Performance targets, scalability strategies, reliability guarantees, observability framework
- **Implementation Plan**: 6-phase roadmap (foundation, core services, data management, testing, pilot, migration)
- **Operational Considerations**: Deployment strategy (blue-green), runbooks, incident response, cost breakdown

### 5.2 Architecture Decision Records
**Document**: [ADRs - Proposal 1](./supporting-documents/adrs-proposal-1.md)

Consolidated ADRs documenting 10 critical architectural decisions:

1. **Event-Driven Architecture Pattern** - Why events over request-response; trade-offs of decoupling vs complexity
2. **Azure Service Bus Topics vs Queues** - Fan-out capability and flexible routing with topic filters
3. **Synchronous vs Asynchronous API** - Supporting both patterns for flexibility; correlation handling
4. **Cosmos DB as Primary Data Store** - Global distribution for GDPR compliance; partition key strategy
5. **Cache-Aside Pattern for Preferences** - In-memory caching saves $2,700/month; 5-min TTL trade-offs
6. **Exponential Backoff Retry Strategy** - 1s, 2s, 4s, 8s, 16s delays; circuit breaker thresholds
7. **Multi-Region Deployment** - Active-active US/EU for data residency; consistency challenges
8. **ULID for Tracking IDs** - Time-sortable 26-char IDs vs UUID; timestamp extraction benefits
9. **Handlebars.js for Templates** - Balance of power and safety; precompilation performance
10. **Webhook Callbacks for Status Updates** - Push model reduces polling by 90%; HMAC security

Each ADR includes context, decision, rationale, consequences (positive and negative), and alternatives considered.

### 5.3 Key Technical Specifications

**Event Schemas**
```json
// NotificationSubmitted Event
{
  "eventType": "NotificationSubmitted",
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "priority": "high",
  "channel": "email",
  "recipient": {"customerId": "cust_123", "email": "user@example.com"},
  "template": {"id": "order-confirmation-v2", "version": "2.1.0"},
  "data": {"orderId": "ORD-123", "orderTotal": "$149.99"},
  "correlationId": "req-abc-123"
}

// NotificationStatus Event
{
  "eventType": "NotificationStatus",
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "status": "sent",
  "channel": "email",
  "provider": "sendgrid",
  "providerMessageId": "msg_sg_xyz123",
  "attempt": 1,
  "timestamp": "2025-11-21T10:15:32Z"
}
```

**API Endpoints**
- `POST /api/v1/notifications` - Submit single notification (sync or async)
- `POST /api/v1/notifications/batch` - Submit up to 1000 notifications
- `GET /api/v1/notifications/{trackingId}` - Query status
- `GET /api/v1/preferences/{customerId}` - Get customer preferences
- `PUT /api/v1/preferences/{customerId}` - Update preferences

**Cosmos DB Containers**
- `customer-preferences` - Partition: `/customerId`, 3,000 RU/s, 10 GB
- `notification-tracking` - Partition: `/trackingId`, 5,000 RU/s, 500 GB
- `templates` - Partition: `/templateId`, 500 RU/s, 1 GB
- `audit-logs` - Partition: `/date`, 1,500 RU/s, 1 TB, 90-day TTL

**Scaling Configuration**
- API Gateway: 3-10 pods (HPA at 70% CPU)
- Email Processor: 2-15 pods (HPA at queue depth > 1000)
- SMS Processor: 2-10 pods (HPA at queue depth > 500)
- Push Processor: 2-8 pods (HPA at queue depth > 2000)
- Status Tracker: 3-8 pods (HPA at queue depth > 5000)

### 5.4 Implementation Roadmap

**Phase 1: Foundation** (Weeks 1-4)
- Azure infrastructure setup (AKS, Service Bus, Cosmos DB, APIM, VNET)
- CI/CD pipelines, monitoring, dashboards

**Phase 2: Core Services** (Weeks 5-10)
- API Gateway, Channel Processors, Status Tracker implementation
- Provider integrations (SendGrid, Twilio, Azure NH)

**Phase 3: Data Management** (Weeks 11-13)
- Cosmos DB container design, preference management APIs
- Template management, audit logging

**Phase 4: Testing & Hardening** (Weeks 14-17)
- Unit tests (80% coverage), integration tests, load tests (2,000 msg/sec)
- Chaos engineering, security testing, performance tuning

**Phase 5: Pilot & Migration** (Weeks 18-26)
- Pilot with 1-2 services (Weeks 18-20)
- Migrate 5 medium services (Weeks 21-24)
- Complete migration (Weeks 25-26)

**Total Timeline**: 26 weeks (6 months)
**Team Size**: 3-4 engineers
**Development Cost**: ~$325k
**Monthly Operational Cost**: $93k (optimized to ~$70k with batching/routing)

### 5.5 Migration Strategy

**Gradual Cutover with Zero Downtime**
1. Deploy CNS infrastructure with feature flags (traffic disabled)
2. Pilot with low-volume services, validate metrics
3. Gradually migrate services with rollback capability
4. Decommission old notification code after full migration

**Rollback Plan**: Feature flags in consuming services enable instant revert to direct provider calls.

---

## Next Steps

### For Implementation
1. **Review** the [Architecture Specification](./supporting-documents/architecture-spec-proposal-1.md) for complete technical details
2. **Study** the [ADRs](./supporting-documents/adrs-proposal-1.md) to understand key design decisions and trade-offs
3. **Set up** development environment following Phase 1 infrastructure setup
4. **Begin** Phase 2 implementation with API Gateway and one channel processor (email recommended)

### For Comparison
- Review [Proposal 2 (API Gateway with Smart Routing)](./proposal-2-api-gateway-smart-routing.md) for alternative approach
- Compare cost, complexity, and timeline trade-offs in [README.md](./README.md)
- Review critique documents for both proposals to understand risks
