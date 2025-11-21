# Proposal 2: API Gateway with Smart Routing

## 1. Executive Summary

### Problem Statement
The Customer Notification Service must handle 50M+ notifications daily across multiple channels while ensuring reliability, scalability, and compliance. The key challenges include managing high throughput (2,000 msg/sec peak), coordinating multiple external providers with different rate limits, tracking delivery status across distributed components, and supporting zero-message-loss guarantees during failures.

### Proposed Solution
This proposal uses a **simplified API gateway pattern** with smart routing and aggressive caching to minimize complexity while maximizing performance. The design centers on a stateless API layer that directly enqueues notifications into priority-based Azure Service Bus queues (critical, high, normal, low). A pool of worker processors pulls from these queues, intelligently routes to the appropriate channel handler, and manages provider interactions. The architecture emphasizes simplicity, predictability, and operational ease over pure event-driven flexibility.

The key differentiator is the **smart routing layer** that sits between the queue and channel handlers. This layer maintains in-memory state about provider health, rate limits, and performance characteristics, making real-time decisions about which notifications to process immediately, which to batch, and which to delay. Customer preferences are cached aggressively in a distributed Redis cache (Azure Cache for Redis) to avoid database lookups on the hot path. This reduces Cosmos DB RU requirements by 70-80% and cuts API latency in half compared to database-first approaches.

The architecture is fundamentally **request-response** with queues serving as durability and buffering, rather than the primary communication pattern. This makes the system easier to reason about, debug, and operate, at the cost of slightly tighter coupling between components.

### Key Benefits
- **Operational simplicity**: Traditional queue-worker pattern that most engineers understand; straightforward debugging with clear request paths
- **Predictable performance**: Direct queue-to-worker flow eliminates event fan-out overhead; 30-40% lower latency than event-driven approaches
- **Cost efficiency**: Aggressive caching reduces Cosmos DB costs by 70%; simpler topology means fewer Service Bus topics/subscriptions
- **Fast time-to-market**: Proven patterns and minimal architectural novelty accelerate development; can deliver in 3-4 months vs 5+ for event-driven
- **Easy migration**: Consuming services simply swap provider URLs for CNS API; wrapper libraries make it transparent

## 2. High-Level Architecture

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Consuming Services                                  │
│                 (Order Service, Account Service, etc.)                       │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ REST API (sync or async)
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Azure API Management (APIM)                             │
│         Authentication, Rate Limiting, Request Validation, Caching           │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CNS API Service (AKS - 5 pods)                       │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ • Validate request payload and assign tracking ID                   │   │
│  │ • Check preferences: Redis cache → Cosmos DB fallback               │   │
│  │ • Apply opt-out and quiet hours rules                               │   │
│  │ • Enqueue to priority-specific Service Bus queue                    │   │
│  │ • Write initial tracking record to Cosmos DB                        │   │
│  │ • Return tracking ID (async mode) OR wait for sent status (sync)   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                    ┌────────────┴──────────────┬─────────────┬──────────────┐
                    │                           │             │              │
                    ▼                           ▼             ▼              ▼
    ┌────────────────────────┐  ┌────────────────────────┐  ┌──────┐  ┌──────┐
    │  Queue: critical       │  │  Queue: high           │  │normal│  │ low  │
    │  (5 sec SLA)           │  │  (30 sec SLA)          │  │(5min)│  │(1hr) │
    └───────────┬────────────┘  └───────────┬────────────┘  └───┬──┘  └───┬──┘
                │                           │                   │         │
                └───────────────────────────┴───────────────────┴─────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              Smart Routing Worker Pool (AKS - 10-30 pods)                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  Each Worker:                                                        │   │
│  │  1. Pull message from priority queues (critical first)              │   │
│  │  2. Check Redis for rendered template (cache hit ~60%)              │   │
│  │  3. If miss: render template and cache result                       │   │
│  │  4. Consult Smart Router for channel decision:                      │   │
│  │     - Check provider health (circuit breaker status)                │   │
│  │     - Check rate limit budget (token bucket)                        │   │
│  │     - Decide: send now, batch, or delay                             │   │
│  │  5. Call appropriate channel handler                                │   │
│  │  6. Update tracking record in Cosmos DB                             │   │
│  │  7. Trigger webhook if configured                                   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Smart Router (in-memory shared state):                             │    │
│  │  • Provider health tracking (last 100 requests per provider)        │    │
│  │  • Rate limit enforcement (token bucket per provider)               │    │
│  │  • Batch accumulation (email batches up to 100 recipients)          │    │
│  │  • Circuit breaker state (open/half-open/closed per provider)       │    │
│  │  • Performance metrics (latency histogram per channel)              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└────────────────────┬────────────────────┬────────────────────┬──────────────┘
                     │                    │                    │
                     ▼                    ▼                    ▼
        ┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
        │  Email Handler     │ │   SMS Handler      │ │   Push Handler     │
        ├────────────────────┤ ├────────────────────┤ ├────────────────────┤
        │ • Batch up to 100  │ │ • No batching      │ │ • Batch up to 1000 │
        │ • Call SendGrid    │ │ • Call Twilio      │ │ • Call Azure NH    │
        │ • Parse response   │ │ • Parse response   │ │ • Parse response   │
        │ • Return status    │ │ • Return status    │ │ • Return status    │
        └────────────────────┘ └────────────────────┘ └────────────────────┘
                     │                    │                    │
                     └────────────────────┴────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│            Azure Cache for Redis (Premium, 26 GB, Multi-AZ)                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ • Customer preferences (cache-aside, 5-min TTL)                     │   │
│  │ • Rendered templates (cache-aside, 1-hour TTL)                      │   │
│  │ • Rate limit counters (sliding window)                              │   │
│  │ • Session state for sync API requests                               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Azure Cosmos DB (Multi-Region)                          │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ • Customer preferences (authoritative source, cache-aside pattern)  │   │
│  │ • Notification tracking (status, attempts, timestamps)              │   │
│  │ • Templates (with versioning)                                       │   │
│  │ • Audit logs (90-day retention, write-optimized)                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                      External Notification Providers                         │
│        SendGrid (Email)  │  Twilio (SMS)  │  Azure Notification Hubs        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Component Overview

**Azure API Management (APIM)**
- Unified entry point for all consuming services
- OAuth 2.0 authentication and per-service rate limiting
- Response caching for status queries (5-second cache)
- Request validation and payload size enforcement
- TLS termination and request/response logging

**CNS API Service**
- Stateless REST API running on AKS (5 pods, auto-scale to 10)
- Validates requests, assigns tracking IDs (ULID format for time-ordering)
- Preference lookup: Redis cache first (P95 < 5ms), Cosmos DB fallback (P95 < 50ms)
- Checks opt-out status, quiet hours, and invalid contact info
- Enqueues to priority-specific Service Bus queue
- For sync requests: stores correlation ID in Redis and subscribes to status updates

**Azure Service Bus Queues**
- Four priority-based queues (critical, high, normal, low)
- Critical queue: 5-second target latency, max 10k messages
- High queue: 30-second target latency, max 50k messages
- Normal queue: 5-minute target latency, max 500k messages
- Low queue: 1-hour target latency, max 2M messages
- Automatic dead-lettering after 5 failed delivery attempts

**Smart Routing Worker Pool**
- Horizontally scalable workers (10-30 pods based on queue depth)
- Pull messages from queues with priority ordering (critical first)
- Consult Smart Router for intelligent processing decisions
- Handle template rendering with cache-first strategy
- Call channel handlers and update tracking records
- Implement exponential backoff retry logic (1s, 2s, 4s, 8s, 16s)

**Smart Router (Embedded in Workers)**
- Shared in-memory state synchronized via Redis pub/sub
- Circuit breaker per provider (open after 10 consecutive failures, 30s timeout)
- Token bucket rate limiting (respects provider limits with 10% safety margin)
- Batching logic: accumulate emails up to 100 or 5 seconds, whichever first
- Provider health scoring based on recent latency and error rates
- Automatic channel fallback (email fails → try SMS if configured)

**Channel Handlers**
- Thin wrappers around provider SDKs (SendGrid, Twilio, Azure NH)
- Email handler: supports batching, processes bounce/complaint webhooks
- SMS handler: no batching, handles carrier-specific errors
- Push handler: supports multicast to device groups
- Parse provider responses and normalize status codes

**Azure Cache for Redis**
- Premium tier (26 GB, multi-AZ for 99.9% SLA)
- Cache-aside pattern for customer preferences (5-min TTL)
- Rendered template cache (1-hour TTL, ~60% hit rate)
- Rate limit counters (sliding window, 1-sec granularity)
- Synchronous request correlation state (30-sec TTL)

**Azure Cosmos DB**
- Multi-region deployment (US West, EU West) for data residency
- Customer preferences: partitioned by `customerId`, read-heavy workload
- Notification tracking: partitioned by `trackingId`, write-heavy workload
- Templates: partitioned by `templateId`, read-heavy workload
- Audit logs: partitioned by `date`, append-only, auto-archived after 90 days

### Data Flow

**Normal Flow (Asynchronous)**
1. Consuming service POSTs notification request to APIM
2. APIM validates auth, checks rate limit, forwards to API Service
3. API Service checks preferences (Redis → Cosmos DB), assigns tracking ID
4. API Service enqueues message to priority-specific queue, returns tracking ID (200 OK)
5. Worker pulls message from queue (critical first, round-robin within priority)
6. Worker consults Smart Router to check provider health and rate limits
7. Worker calls channel handler (batches if applicable)
8. Handler invokes provider API and returns status
9. Worker updates tracking record in Cosmos DB
10. If webhook configured: worker triggers callback (fire-and-forget)

**Synchronous Flow**
1-4. Same as asynchronous flow, but API Service stores correlation ID in Redis
5-9. Same as asynchronous flow
10. Worker publishes status to Redis pub/sub channel (correlation ID)
11. API Service receives notification via Redis subscription
12. API Service returns status to caller (200 OK with delivery confirmation)

### Integration Points

- **External Providers**: SendGrid (email, batch API), Twilio (SMS, single API), Azure Notification Hubs (push, multicast API)
- **Internal Services**: 15+ consuming services via REST API through APIM
- **Azure AD**: OAuth 2.0 client credentials flow for service-to-service auth
- **Application Insights**: Telemetry, distributed tracing (W3C Trace Context standard)
- **Azure Monitor**: Metrics dashboards, alerting rules, action groups

## 3. Core Trade-offs

### Key Advantages
1. **Operational simplicity**: Queue-worker pattern is well-understood; debugging follows clear request paths; fewer "moving parts" than event-driven
2. **Lower latency**: Cache-first strategy reduces P95 latency to < 150ms (vs 200ms requirement); direct queue processing eliminates event fan-out overhead
3. **Cost efficiency**: Redis caching cuts Cosmos DB RU needs by 70% (~$4,000/month savings); simpler Service Bus topology saves ~$500/month
4. **Easier testing**: Request-response flows are straightforward to unit test; integration tests don't require complex event mocking
5. **Faster development**: Proven patterns accelerate delivery; less architectural risk than event-driven approaches

### Key Disadvantages
1. **Tighter coupling**: Workers directly call channel handlers; changes to handlers may impact workers; harder to insert new components in the processing pipeline
2. **Limited flexibility**: Adding new channels requires code changes to Smart Router; not as naturally extensible as event-driven pub/sub
3. **In-memory state risks**: Smart Router state in worker pods must be synchronized; pod restarts lose rate limit counters (temporary over-sending risk)
4. **Batch coordination**: Email batching requires coordination across workers; complex logic to balance batch size vs latency
5. **Scaling granularity**: Workers handle all channels; can't independently scale email processing vs SMS processing

### Risk Summary
- **Redis single point of failure**: Cache miss storm could overwhelm Cosmos DB; mitigate with Redis clustering, circuit breaker to database, and request coalescing
- **Worker state synchronization**: Inconsistent Smart Router state could violate rate limits; mitigate with Redis pub/sub for state sharing and conservative limits
- **Queue head-of-line blocking**: Slow email batch could delay SMS processing; mitigate with separate worker pools per channel or priority-based preemption
- **Synchronous request scalability**: Long-polling for sync requests holds connections; mitigate with connection limits and aggressive timeouts (< 5 sec)

## 4. Feasibility Assessment

### Implementation Complexity: **Medium**

**Justification**: This architecture uses familiar patterns (REST API, queues, workers) that most engineers understand. The main complexity is in the Smart Router logic (rate limiting, batching, circuit breakers), but these are well-documented patterns with proven libraries. Redis caching is straightforward with cache-aside pattern. The synchronous API pattern adds some complexity (Redis pub/sub for correlation), but it's optional for initial launch.

**Development effort**:
- API Service: 2-3 weeks
- Worker pool and Smart Router: 4-5 weeks (includes rate limiting, batching, circuit breaker logic)
- Channel handlers: 1 week each (can parallelize)
- Redis caching layer: 1 week
- Infrastructure setup: 1-2 weeks
- Testing and debugging: 3 weeks
- **Total: ~3-4 months with 3-4 engineers**

### Operational Impact

**Changes required**:
- Operations team must monitor queue depths, worker scaling, and Redis health
- Debugging focuses on worker logs and Redis cache hit rates
- Incident response: check Smart Router circuit breaker states, rate limit counters
- New runbooks for cache invalidation, queue draining, and worker restarts

**Benefits**:
- Straightforward request path makes debugging faster
- Redis caching reduces database load (easier to maintain Cosmos DB SLA)
- Fewer components than event-driven architecture (simpler on-call)
- Clear scaling triggers: queue depth → add workers, cache miss rate → add Redis memory

### Migration Approach

**Strategy: Wrapper library with feature flags**

1. **Phase 0** (Week 1-2): Publish wrapper library for consuming services that abstracts provider SDKs; consuming services adopt library in their existing code paths
2. **Phase 1** (Month 1-2): Deploy CNS infrastructure; wrapper library defaults to direct provider calls (no behavior change)
3. **Phase 2** (Month 3): Enable wrapper library routing to CNS for 1-2 pilot services (~1M/day); validate performance and cost
4. **Phase 3** (Month 4): Roll out to 5 medium-volume services (~20M/day); tune Redis caching and worker scaling
5. **Phase 4** (Month 5-6): Migrate remaining services; flip global feature flag to route 100% traffic through CNS
6. **Rollback plan**: Feature flag in wrapper library instantly reverts to direct provider calls; no code changes needed

**Zero-downtime approach**: Wrapper library enables gradual rollout and instant rollback via feature flags. Consuming services never experience downtime.

### Rough Order of Magnitude

**Development costs**:
- Engineering: 3.5 engineers × 4 months × $15k/month = $210k
- Wrapper library development: 1 engineer × 2 months × $15k/month = $30k
- Infrastructure (pre-production): $3k/month × 4 months = $12k
- **Total development: ~$252k**

**Monthly operational costs** (at 50M/day volume):
- Provider costs: $82,115/month (SendGrid, Twilio, Azure NH)
- Azure infrastructure: $8,200/month breakdown:
  - AKS: $600/month (5 API pods + 15 workers avg)
  - Service Bus Standard: $700/month (4 queues, no Premium needed)
  - Cosmos DB: $4,100/month (7,000 RU/s due to caching; 40% reduction)
  - Redis Premium: $1,200/month (26 GB, multi-AZ)
  - APIM Premium: $2,800/month
  - Monitoring: $400/month
- **Total: $90,315/month** ($2,830/month savings vs event-driven proposal)

**Cost optimization potential**: With intelligent batching and channel routing, projected savings of $18-25k/month bring costs to ~$67-72k/month (cost-neutral vs current $70k/month distributed costs).

**At 150M/day (3x growth)**:
- Provider costs: ~$246k/month (linear scaling)
- Infrastructure: ~$14k/month (Cosmos DB and Redis scale; AKS grows modestly; Service Bus unchanged)
- **Total: ~$260k/month** ($1.73 per 1000 notifications; slightly cheaper than event-driven at scale)

---

## 5. Detailed Design Documentation

This proposal has been refined into comprehensive, implementation-ready documentation. The following supporting documents provide detailed technical specifications:

### 5.1. Architecture Specification

📄 **[Architecture Specification for Proposal 2](./supporting-documents/architecture-spec-proposal-2.md)**

The architecture specification provides comprehensive technical details including:
- **Executive Summary**: Key architectural highlights and design principles
- **System Overview**: Purpose, scope, goals, and success criteria
- **Architecture Overview**: Complete component diagram, technology stack, and design patterns
- **Detailed Design**: Component-level specifications including:
  - API Service with cache-first preference lookup
  - Smart Routing Workers with 13-step processing flow
  - Channel Handlers (Email, SMS, Push) with provider-specific logic
  - Smart Router algorithms (circuit breaker, token bucket, batching)
- **Data Architecture**: Data models, caching strategy, partitioning approach
- **API Specifications**: Complete request/response schemas for all endpoints
- **Security Architecture**: Authentication, encryption, secrets management, network security
- **Non-Functional Requirements**: Performance targets, scalability limits, reliability mechanisms
- **Implementation Plan**: 5-phase roadmap over 20 weeks
- **Operational Considerations**: Deployment strategy, monitoring, cost analysis

### 5.2. Architecture Decision Records (ADRs)

📄 **[Consolidated ADRs for Proposal 2](./supporting-documents/adrs-proposal-2.md)**

Key architectural decisions documented with full context and rationale:

1. **Request-Response Pattern with Queue Buffering** - Chose queues over events for operational simplicity
2. **Azure Service Bus Standard vs Premium** - Standard tier sufficient; saves $640/month
3. **Redis for Aggressive Caching** - 80% cache hit rate saves $1,500/month net
4. **Smart Router for Intelligent Batching and Rate Limiting** - Embedded logic for cost optimization
5. **Priority-Based Queues** - Four separate queues for SLA enforcement
6. **Cosmos DB with Reduced RU Provisioning** - 30% reduction enabled by Redis caching
7. **Synchronous API with Redis Pub/Sub** - Low-latency status updates for critical notifications
8. **Wrapper Library for Migration** - Feature-flag driven migration with zero-downtime rollback
9. **Same Provider Selection and Template Engine** - Consistent with Proposal 1
10. **Multi-Region Deployment for GDPR Compliance** - Active-active US/EU deployment

Each ADR includes: Status, Context, Decision, Rationale, Consequences (positive and negative), and Alternatives Considered.

### 5.3. Key Technical Specifications

**Smart Router Configuration**
```yaml
circuit_breaker:
  failure_threshold: 10              # Open after 10 consecutive failures
  timeout_duration: 30s              # Stay open for 30 seconds
  half_open_success_threshold: 3    # Close after 3 consecutive successes

token_bucket_rate_limits:
  sendgrid:
    capacity: 9000                   # 90% of 10k/sec provider limit
    refill_rate: 9000                # Tokens per second
  twilio:
    capacity: 900                    # 90% of 1k/sec provider limit
    refill_rate: 900
  azure_nh:
    capacity: 9000                   # 90% of 10k/sec provider limit
    refill_rate: 9000

batching:
  email:
    max_batch_size: 100              # SendGrid batch limit
    max_wait_time: 5s                # Max time to accumulate batch
  push:
    max_batch_size: 1000             # Azure NH multicast limit
    max_wait_time: 5s
  sms:
    batching_enabled: false          # Twilio doesn't support batching
```

**Redis Caching Configuration**
```yaml
cache_patterns:
  preferences:
    ttl: 300s                        # 5-minute TTL
    pattern: "pref:{customerId}"
    expected_hit_rate: 80%
  
  templates:
    ttl: 3600s                       # 1-hour TTL
    pattern: "tmpl:{templateId}:{version}"
    expected_hit_rate: 60%
  
  rate_limiters:
    ttl: 1s                          # 1-second granularity
    pattern: "rate:{service}:{window}"
    sliding_window: true
  
  sync_correlation:
    ttl: 30s                         # Sync request timeout
    pattern: "sync:{correlationId}"
    pubsub_channel: "status:*"
```

**Priority Queue Configuration**
```yaml
service_bus_queues:
  critical:
    max_size: 1GB
    lock_duration: 30s
    max_delivery_count: 5
    default_ttl: 5m
    target_latency: 5s
    
  high:
    max_size: 5GB
    lock_duration: 60s
    max_delivery_count: 5
    default_ttl: 1h
    target_latency: 30s
    
  normal:
    max_size: 10GB
    lock_duration: 5m
    max_delivery_count: 5
    default_ttl: 24h
    target_latency: 5m
    
  low:
    max_size: 10GB
    lock_duration: 10m
    max_delivery_count: 3
    default_ttl: 7d
    target_latency: 1h
```

**Worker Scaling Configuration**
```yaml
horizontal_pod_autoscaler:
  api_service:
    min_replicas: 5
    max_replicas: 10
    target_cpu_utilization: 70%
    target_memory_utilization: 80%
  
  worker_pool:
    min_replicas: 10
    max_replicas: 30
    custom_metrics:
      - name: service_bus_queue_depth
        target_value: 100              # Scale up if any queue > 100 messages
      - name: redis_cache_hit_rate
        target_value: 75%              # Scale up if cache hit rate < 75%
```

### 5.4. Implementation Roadmap

**Phase 1: Foundation (Weeks 1-4)**
- Set up Azure infrastructure (AKS, Service Bus, Cosmos DB, Redis, APIM)
- Implement API Service with basic request validation
- Implement cache-aside pattern for preferences
- Deploy monitoring and alerting (Application Insights, Azure Monitor)

**Phase 2: Core Workers (Weeks 5-8)**
- Implement worker pool with priority queue polling
- Implement Smart Router logic (circuit breaker, token bucket)
- Implement channel handlers (Email, SMS, Push)
- Implement retry logic with exponential backoff

**Phase 3: Advanced Features (Weeks 9-12)**
- Implement email batching logic in Smart Router
- Implement Redis pub/sub for synchronous API pattern
- Implement template rendering with cache
- Implement webhook callbacks for status updates

**Phase 4: Migration Support (Weeks 13-16)**
- Develop wrapper libraries for consuming services (C#, Java, Python, Node.js)
- Implement feature flag system for gradual rollout
- Deploy pilot services (1-2 low-volume services)
- Performance tuning and optimization

**Phase 5: Production Rollout (Weeks 17-20)**
- Migrate 5 medium-volume services (~20M/day)
- Migrate remaining services to CNS
- Flip global feature flag to 100% CNS routing
- Decommission direct provider integrations

**Total Timeline**: 20 weeks (~5 months) with 3-4 engineers

### 5.5. Migration Strategy

**Wrapper Library Approach**

The migration uses a **wrapper library with feature flags** to enable zero-downtime migration and instant rollback:

```javascript
// Example: Consuming service using wrapper library
const notificationClient = new NotificationClient({
  cnsApiUrl: 'https://cns-api.company.com',
  sendGridApiKey: process.env.SENDGRID_API_KEY,
  twilioCredentials: { /* ... */ },
  featureFlagService: featureFlagClient
});

// Wrapper library internally routes based on feature flag
await notificationClient.sendEmail({
  recipient: user.email,
  template: 'order-confirmation',
  data: { orderId: '12345' },
  priority: 'high'
});
```

**Migration Phases**:

1. **Adopt wrapper library** (no behavior change) - Services update to use wrapper library but feature flag routes to direct provider calls
2. **Enable CNS for pilot services** (1-2 low-volume) - Flip feature flag for pilot services; validate performance
3. **Gradual rollout** (5 medium-volume services) - Roll out to more services; monitor cache hit rates and latency
4. **Full migration** (all services) - Flip global feature flag to route 100% traffic through CNS
5. **Instant rollback capability** - If issues detected, flip feature flag back to direct provider calls (no code changes or deployments)

**Rollback Guarantee**: Feature flag toggle reverts to direct provider calls in < 5 seconds (no code deployment required).

---

## Next Steps

This proposal is now fully refined with implementation-ready specifications:

1. **Review Architecture Specification**: Examine component designs, API contracts, and security architecture
2. **Review ADRs**: Understand key design decisions and trade-offs
3. **Compare with Proposal 1**: See [comparison matrix](./supporting-documents/comparison-matrix.md) for side-by-side evaluation
4. **Review Critiques**: See [critique documents](./supporting-documents/) for expert analysis of both proposals
5. **Make Go/No-Go Decision**: Select this proposal, Proposal 1, or request modifications
6. **Begin Implementation**: Follow the 5-phase roadmap (20 weeks) with assigned engineering team

For questions or clarifications about this proposal, refer to the detailed architecture specification and ADR documents above
