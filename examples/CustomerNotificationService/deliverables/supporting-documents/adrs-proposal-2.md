# Architecture Decision Records (ADRs) - Proposal 2: API Gateway with Smart Routing

**Version:** 1.0  
**Date:** November 21, 2025  
**Proposal:** API Gateway with Smart Routing for Customer Notification Service

---

## ADR-001: Request-Response Pattern with Queue Buffering

**Status:** Accepted

**Context**

The Customer Notification Service must handle high throughput (50M+ notifications/day) while remaining operationally simple for the engineering team. We evaluated two primary architectural communication patterns:
1. **Event-driven architecture** with topics and subscriptions (loose coupling, eventual consistency)
2. **Request-response architecture** with queues for buffering (tighter coupling, operational simplicity)

The event-driven approach (Proposal 1) provides excellent decoupling and scalability but adds complexity with distributed tracing, eventual consistency, and multiple event flows. The request-response pattern is more familiar to engineers but traditionally suffers from tight coupling.

**Decision**

Use **request-response pattern with Azure Service Bus queues** as durability buffers, not as the primary communication mechanism. The API service directly enqueues messages, workers directly dequeue and process, and status updates go directly to Cosmos DB (not through events).

**Rationale**

- **Operational simplicity**: Traditional queue-worker pattern that most engineers understand; straightforward debugging with clear request paths
- **Predictable performance**: Direct queue-to-worker flow eliminates event fan-out overhead; 30-40% lower latency
- **Easier testing**: Request-response flows are straightforward to unit test without event simulation
- **Faster time-to-market**: Proven patterns accelerate development (5 months vs 6 months for event-driven)
- **Lower complexity**: Fewer moving parts (no topic subscriptions, fewer Service Bus resources)

**Consequences**

*Positive:*
- Debugging is straightforward: Request → Queue → Worker → Provider (linear flow)
- Lower latency: No event publishing/subscription overhead (P95 API latency 150ms vs 200ms)
- Familiar patterns: Most engineers have experience with queue-worker systems
- Simpler onboarding: New team members productive faster
- Lower infrastructure costs: Standard Service Bus sufficient ($700/month vs $1,340/month Premium)

*Negative:*
- **Tighter coupling**: Workers directly integrated with channel handlers; changes may require coordination
- **Less flexible**: Adding new channels requires code changes to workers (not just new subscriptions)
- **Shared worker pool**: All channels share the same worker pool; can't independently scale email vs SMS vs push processors
- **Limited extensibility**: Harder to insert new processing stages without modifying worker code

**Alternatives Considered**

*Event-Driven Architecture (Proposal 1)*
- Better decoupling, independent channel scaling, natural audit trail
- **Rejected because:** Higher complexity, longer development time, harder debugging, higher costs

*Hybrid (Sync API + Async Processing)*
- Combines synchronous API with asynchronous backend
- **Rejected because:** Adds complexity without significant benefits; still requires event handling for status updates

---

## ADR-002: Azure Service Bus Standard vs Premium

**Status:** Accepted

**Context**

Azure Service Bus offers two tiers:
- **Standard**: $0.05 per million operations, 256 KB message size, shared capacity
- **Premium**: Fixed cost ($670/messaging unit/month), dedicated resources, guaranteed throughput (8 MB/sec per unit)

Our requirements:
- 580 msg/sec average (2,000 msg/sec peak) = ~50M messages/day
- Message size: ~2 KB (notification metadata)
- Need for guaranteed throughput and predictable latency

**Decision**

Use **Azure Service Bus Standard tier** with four separate queues (critical, high, normal, low) instead of Premium tier.

**Rationale**

- **Cost efficiency**: Standard tier costs ~$700/month for our volume vs $1,340/month for Premium (2 messaging units)
- **Sufficient throughput**: Standard tier can handle our peak 2,000 msg/sec with 2 KB messages
- **No need for topics**: We use direct queues, not topics/subscriptions, so Standard is sufficient
- **Predictable costs**: Standard pricing is pay-per-operation (easy to forecast)
- **Good enough latency**: Standard tier provides < 100ms latency (acceptable for our use case)

**Consequences**

*Positive:*
- **Cost savings**: $640/month ($7,680/year) vs Premium tier
- Simpler configuration: No need for messaging units or throughput tuning
- Standard tier scales automatically with demand
- Still provides message durability and dead-lettering

*Negative:*
- No guaranteed throughput (shared capacity with other tenants)
- No geo-disaster recovery (Premium feature)
- No advanced filtering (but we don't need it with direct queues)
- Slightly higher latency variability than Premium (P99 can spike during Azure issues)

**Risk Mitigation**

- If Standard tier throttles at peak, we can upgrade to Premium (no code changes required)
- Monitor queue depth and message latency; alert if consistent throttling detected
- Design workers to handle queue backlogs gracefully (auto-scaling based on queue depth)

**Alternatives Considered**

*Service Bus Premium (Proposal 1)*
- Guaranteed throughput, lower latency variability, geo-DR
- **Rejected because:** 95% higher cost without significant benefits for our workload; Standard tier meets our SLA targets

*Azure Storage Queues*
- Even cheaper ($0.00036 per 10,000 operations)
- **Rejected because:** No built-in dead-letter queue; no message locking (at-most-once vs at-least-once); no FIFO guarantees

---

## ADR-003: Redis for Aggressive Caching

**Status:** Accepted

**Context**

Customer preferences are read on every notification request (580/sec average, 2,000/sec peak). Without caching:
- Cosmos DB cost: 580 req/sec × 10 RU = 5,800 RU/s = ~$3,400/month just for preference reads
- P95 latency: ~50ms per read (adds to API latency)

Caching options:
1. **In-memory caching per pod** (Proposal 1): Simple, no external dependency, but cache invalidation complex
2. **Distributed Redis cache**: Shared cache across pods, atomic operations, pub/sub for state sharing
3. **No caching**: Always read from Cosmos DB (simplest but expensive and slow)

**Decision**

Use **Azure Cache for Redis Premium** (26 GB, multi-AZ) as a distributed cache layer for customer preferences, rendered templates, rate limit counters, and Smart Router state.

**Rationale**

- **Massive cost savings**: 80%+ cache hit rate reduces Cosmos DB reads by 80% = ~$2,700/month savings (Redis costs $1,200/month = net savings $1,500/month)
- **Improved latency**: Redis read < 5ms vs Cosmos DB 50ms (reduces P95 API latency by ~40ms)
- **Shared cache**: All pods see same cache state (no cache invalidation complexity)
- **Atomic operations**: Redis INCR for rate limiting, SET EX for TTL-based cache
- **Pub/Sub**: Synchronize Smart Router state across worker pods in real-time
- **Multi-AZ**: Redis Premium provides 99.9% SLA (acceptable reliability)

**Consequences**

*Positive:*
- **$1,500/month net savings** ($2,700 DB savings - $1,200 Redis cost)
- **40ms latency improvement** for API requests (cache hit scenario)
- Consistent cache across all pods (no stale reads)
- Redis pub/sub enables real-time state synchronization for Smart Router
- Atomic operations simplify rate limiting and counters

*Negative:*
- **External dependency**: Redis unavailable → fallback to Cosmos DB (acceptable but slower)
- **Cache staleness**: 5-min TTL means preference updates take up to 5 min to propagate (acceptable per business requirements)
- **Additional operational overhead**: Must monitor Redis health, memory usage, eviction rate
- **Single point of failure** (mitigated by multi-AZ deployment and fallback to DB)

**Cache-Aside Pattern Implementation**

```
Read path:
1. Check Redis for key
2. If found: return cached value (cache hit)
3. If not found: query Cosmos DB, store in Redis with TTL, return value (cache miss)

Write path:
1. Update Cosmos DB (source of truth)
2. Delete Redis key (invalidate cache)
3. Next read will cache miss and refresh from DB
```

**Alternatives Considered**

*In-Memory Caching per Pod*
- No external dependency, faster (no network)
- **Rejected because:** Complex cache invalidation (must notify all pods); cache inconsistency during pod scale-up/down; no atomic operations for rate limiting

*No Caching*
- Simplest implementation, always consistent
- **Rejected because:** High Cosmos DB costs ($3,400/month extra); adds 50ms to every API request (violates latency SLA)

---

## ADR-004: Smart Router for Intelligent Batching and Rate Limiting

**Status:** Accepted

**Context**

Notification providers have different characteristics:
- **SendGrid**: 10,000 emails/sec limit, supports batching (100 recipients/request)
- **Twilio**: 1,000 SMS/sec limit, no batching (1 recipient/request)
- **Azure NH**: 10,000 push/sec limit, supports multicast (1,000 devices/request)

Without intelligent routing:
- We could overwhelm providers during traffic spikes
- We'd miss batching opportunities (send 100 emails as 100 separate requests)
- We'd struggle to handle provider outages gracefully

Options for managing provider interactions:
1. **Simple workers**: Each worker independently calls providers (no coordination)
2. **Smart Router**: Centralized logic for rate limiting, batching, circuit breaking
3. **Separate rate limiter service**: Dedicated service for rate limiting

**Decision**

Implement **Smart Router as embedded logic within workers** with in-memory state synchronized across pods via Redis pub/sub. The Smart Router manages:
- Provider health tracking (circuit breaker pattern)
- Rate limiting (token bucket algorithm)
- Batching logic (accumulate and flush strategies)
- Channel fallback (email fails → retry via SMS if configured)

**Rationale**

- **Cost optimization**: Batching reduces API calls to providers (100 emails as 1 batch vs 100 individual requests)
- **Provider protection**: Token bucket rate limiting prevents overwhelming providers during bursts
- **Resilience**: Circuit breaker stops sending to failing providers, allowing recovery time
- **Flexibility**: Can adjust batching and rate limiting per provider without infrastructure changes
- **Performance**: Embedded logic avoids network hops to separate rate limiter service

**Consequences**

*Positive:*
- **Email batching**: Reduces SendGrid API calls by 99% (100 emails as 1 request)
- **Rate limit compliance**: Never exceed provider limits (10% safety margin)
- **Graceful degradation**: Circuit breaker prevents cascading failures
- **Intelligent fallback**: Email fails → try SMS (if configured)
- **Real-time adaptation**: Smart Router reacts to provider health changes

*Negative:*
- **Increased worker complexity**: Workers must embed Smart Router logic (more code to maintain)
- **State synchronization overhead**: Redis pub/sub adds ~5ms latency for state updates
- **Testing complexity**: Must test batching, rate limiting, and circuit breaker logic
- **Potential state inconsistency**: Pod restarts lose in-memory state (temporary rate limit violations possible)

**Smart Router Components**

*Token Bucket Rate Limiter:*
- Capacity: 90% of provider limit (e.g., 9,000 tokens/sec for SendGrid)
- Refill rate: Same as capacity (9,000 tokens/sec)
- Logic: Acquire tokens before sending; delay if bucket empty

*Circuit Breaker:*
- States: CLOSED (normal), OPEN (failing), HALF_OPEN (testing recovery)
- Trigger: 10 consecutive failures → OPEN
- Timeout: 30 seconds before HALF_OPEN
- Recovery: 3 consecutive successes → CLOSED

*Batch Accumulator:*
- Email: Accumulate up to 100 recipients OR 5 seconds (whichever first)
- Push: Accumulate up to 1,000 devices OR 5 seconds
- SMS: No batching (Twilio doesn't support it)

**Alternatives Considered**

*Simple Workers (No Smart Router)*
- Each worker independently calls providers
- **Rejected because:** No rate limiting coordination (could overwhelm providers); misses batching opportunities; no circuit breaker protection

*Separate Rate Limiter Service*
- Dedicated microservice manages rate limits
- **Rejected because:** Adds network latency (extra service hop); increases complexity; single point of failure

---

## ADR-005: Priority-Based Queues

**Status:** Accepted

**Context**

Notifications have varying urgency levels:
- **Critical** (5%): Security alerts, account lockouts (must send within 5 seconds)
- **High** (20%): Order confirmations, password resets (target 30 seconds)
- **Normal** (60%): Shipping updates, newsletters (target 5 minutes)
- **Low** (15%): Recommendations, marketing (target 1 hour)

With a single queue (FIFO), critical notifications could be delayed behind millions of low-priority messages during traffic spikes (head-of-line blocking).

Priority handling options:
1. **Single queue with priority metadata**: Workers prioritize messages programmatically
2. **Separate queues per priority**: Workers poll queues in priority order
3. **Service Bus topics with priority filters**: Event-driven routing (Proposal 1)

**Decision**

Use **four separate Azure Service Bus queues** (critical, high, normal, low). Workers poll queues in priority order: critical first, then high, then normal, then low.

**Rationale**

- **Guaranteed priority**: Critical messages processed first even during backlogs
- **Clear SLA enforcement**: Each queue has explicit SLA (5s, 30s, 5m, 1h)
- **Independent scaling**: Can scale workers based on critical queue depth (weighted)
- **Operational visibility**: Separate queue metrics make it obvious if critical messages are backing up
- **Simple implementation**: Standard Service Bus queues (no need for Premium topics)

**Consequences**

*Positive:*
- **SLA compliance**: Critical notifications never delayed by low-priority traffic
- **Clear monitoring**: Separate queue depth metrics per priority
- **Flexible scaling**: Can add more workers when critical queue depth spikes
- **Cost-effective**: Standard tier queues are cheap ($0.05 per million operations)

*Negative:*
- **More queues to monitor**: 4 queues instead of 1 (but better visibility)
- **Potential worker starvation**: If critical queue is always full, normal/low messages could be delayed indefinitely (mitigated by weighted scaling)
- **Higher Service Bus costs**: 4 queues instead of 1 (but still cheap ~$700/month)

**Worker Polling Logic**

```
while (true) {
  // Poll critical queue first
  message = await criticalQueue.receiveMessage(timeout: 1s);
  if (message) {
    processMessage(message);
    continue;
  }
  
  // No critical messages, try high
  message = await highQueue.receiveMessage(timeout: 1s);
  if (message) {
    processMessage(message);
    continue;
  }
  
  // No high messages, try normal
  message = await normalQueue.receiveMessage(timeout: 5s);
  if (message) {
    processMessage(message);
    continue;
  }
  
  // No normal messages, try low
  message = await lowQueue.receiveMessage(timeout: 30s);
  if (message) {
    processMessage(message);
  }
}
```

**Alternatives Considered**

*Single Queue with Priority Metadata*
- Workers programmatically prioritize messages
- **Rejected because:** Head-of-line blocking (low-priority messages still processed before critical messages arrive); complex worker logic; no queue depth visibility per priority

*Service Bus Topics with Priority Filters (Proposal 1)*
- Event-driven routing to priority-specific subscriptions
- **Rejected because:** Requires Premium tier ($1,340/month); adds event overhead; overkill for priority handling

---

## ADR-006: Cosmos DB with Reduced RU Provisioning

**Status:** Accepted

**Context**

Cosmos DB is the authoritative data store for customer preferences, notification tracking, templates, and audit logs. Without caching, preference lookups require 5,800 RU/s (580 req/sec × 10 RU per read), costing ~$3,400/month.

With Redis caching (80%+ hit rate), only 20% of preference lookups hit Cosmos DB:
- 580 req/sec × 20% = 116 req/sec = 1,160 RU/s for preferences
- Plus tracking writes (~2,900 RU/s) and other operations (~900 RU/s)
- Total: ~5,000 RU/s with caching vs ~10,000 RU/s without

**Decision**

Provision **7,000 RU/s** in Cosmos DB (30% reduction from Proposal 1's 10,000 RU/s) and rely on Redis caching to absorb 80%+ of preference reads. Use Cosmos DB autoscale to burst to 21,000 RU/s if cache fails.

**Rationale**

- **70% cost reduction on preferences**: Only pay for cache misses (20% of reads)
- **Lower baseline RU cost**: 7,000 RU/s = $4,100/month vs 10,000 RU/s = $5,840/month (saves $1,740/month)
- **Autoscale safety net**: Can burst to 3x provisioned RU/s if Redis fails (acceptable degradation)
- **Net savings**: $1,740 DB savings - $1,200 Redis cost = $540/month net savings (plus latency improvement)

**Consequences**

*Positive:*
- **$1,740/month savings** on Cosmos DB (vs Proposal 1 without Redis)
- Lower baseline costs during normal operations (cache hit rate 80%+)
- Autoscale protects against cache failure (bursts to 21,000 RU/s automatically)

*Negative:*
- **Redis dependency**: If Redis fails, Cosmos DB throttles at 7,000 RU/s until autoscale kicks in (~30s delay)
- **Higher cache miss cost**: Cache misses during peak could cause throttling (mitigated by autoscale)
- **Complex capacity planning**: Must monitor both Redis hit rate and Cosmos DB RU consumption

**Risk Mitigation**

- Redis Premium multi-AZ provides 99.9% SLA (unlikely to fail)
- Cosmos DB autoscale absorbs traffic spikes when Redis fails
- Circuit breaker to Cosmos DB if throttling detected (serve stale from cache)
- Monitor cache hit rate; alert if drops below 70%

**Alternatives Considered**

*No Caching (Proposal 1 without optimization)*
- Simplest approach, no external dependencies
- **Rejected because:** 70% higher Cosmos DB costs; 50ms higher latency per request

*More aggressive caching (15-min TTL)*
- Even fewer Cosmos DB reads, lower costs
- **Rejected because:** Unacceptable staleness for opt-outs (business requirement is < 24 hours; 15-min TTL acceptable but 5-min is safer)

---

## ADR-007: Synchronous API with Redis Pub/Sub

**Status:** Accepted

**Context**

Some consuming services require synchronous confirmation that notifications were sent (e.g., critical security alerts). With a pure async system, they must poll for status.

Options for supporting synchronous API:
1. **Async only**: Always return tracking ID immediately (no sync support)
2. **Long-polling**: API Gateway waits for worker to complete, polls Cosmos DB for status
3. **Redis pub/sub**: Worker publishes status update to Redis channel, API Gateway subscribes with correlation ID filter

**Decision**

Support **synchronous API pattern via Redis pub/sub** for status updates. API Gateway stores correlation ID in Redis, subscribes to `status-updates` channel with correlation filter, and waits up to 2 seconds for worker to publish status.

**Rationale**

- **Low latency**: Redis pub/sub delivers status updates in < 5ms (vs polling Cosmos DB every 100ms)
- **Efficient**: No repeated database queries; single subscription per request
- **Scalable**: Redis pub/sub handles 100k+ messages/sec
- **Flexible**: Supports both async (immediate response) and sync (wait for status) patterns

**Consequences**

*Positive:*
- Critical notifications get immediate feedback (sync pattern)
- Low overhead: Redis pub/sub is fast and lightweight
- Clean separation: Workers publish once; API Gateway subscribes per request

*Negative:*
- **Redis dependency for sync requests**: If Redis fails, sync requests timeout (acceptable; client can retry as async)
- **Connection overhead**: Each sync request holds API Gateway connection for up to 2 seconds
- **Timeout complexity**: Must handle timeouts gracefully (return 504 Gateway Timeout if worker doesn't respond in 2s)

**Implementation**

```javascript
// API Gateway (sync request)
async function handleSyncRequest(notification) {
  const correlationId = generateCorrelationId();
  const trackingId = generateTrackingId();
  
  // Store correlation in Redis (30s TTL)
  await redis.setex(`sync:${correlationId}`, 30, trackingId);
  
  // Subscribe to status updates channel
  const subscription = redis.subscribe(`status:${correlationId}`);
  
  // Enqueue notification
  await servicebus.send(queue, { ...notification, trackingId, correlationId });
  
  // Wait for status update (max 2 seconds)
  const status = await subscription.receive(timeout: 2000);
  
  if (status) {
    return { trackingId, status: status.status, sentAt: status.timestamp };
  } else {
    return { trackingId, status: 'timeout', message: 'Status not available within 2s' };
  }
}

// Worker (after sending notification)
async function publishStatus(trackingId, correlationId, status) {
  if (correlationId) {
    // Publish to correlation-specific channel for sync requests
    await redis.publish(`status:${correlationId}`, { trackingId, status });
  }
}
```

**Alternatives Considered**

*Long-Polling Cosmos DB*
- API Gateway polls Cosmos DB every 100ms for status
- **Rejected because:** Wastes RU/s on repeated queries; higher latency; more complex

*Async Only (No Sync Support)*
- Simplest implementation, no Redis dependency for sync requests
- **Rejected because:** Critical notifications need immediate feedback; forces clients to implement polling

---

## ADR-008: Wrapper Library for Migration

**Status:** Accepted

**Context**

15+ consuming services currently call SendGrid, Twilio, and Azure NH APIs directly. We need to migrate them to CNS API with zero downtime and instant rollback capability.

Migration strategies:
1. **Big bang**: All services update to call CNS API in one deployment
2. **Service-by-service**: Each service updates independently over 6 months
3. **Wrapper library with feature flags**: Abstract provider calls behind library, flip flag to route through CNS

**Decision**

Publish **wrapper library** that abstracts notification sending behind a unified interface. Wrapper library supports feature flags to route requests either directly to providers (old path) or through CNS API (new path). Consuming services adopt wrapper library first (no behavior change), then we gradually enable CNS routing via feature flags.

**Rationale**

- **Zero downtime**: Feature flag flips routing without code changes or deployments
- **Instant rollback**: If CNS has issues, flip flag back to direct provider calls
- **Gradual migration**: Enable CNS for 1 service at a time, validate, proceed
- **No urgent deadlines**: Services can adopt wrapper library at their own pace
- **Flexibility**: Can roll back individual services if they encounter issues

**Consequences**

*Positive:*
- **Risk-free migration**: Instant rollback capability reduces migration risk
- **Phased approach**: Validate CNS at low volume before high-volume services migrate
- **No code changes for rollback**: Feature flag toggle (no redeploy)
- **Observability**: Can compare metrics (old path vs new path) during migration

*Negative:*
- **Wrapper library maintenance**: Must maintain library for multiple languages (C#, Java, Python, Node.js)
- **Feature flag complexity**: Must manage feature flags across 15+ services
- **Double latency during transition**: Some requests go through CNS, some go direct (mixed metrics)
- **Library adoption effort**: Services must update to use wrapper library (1-2 days per service)

**Wrapper Library Interface (Pseudocode)**

```javascript
// Wrapper library
class NotificationClient {
  constructor(config) {
    this.featureFlags = new FeatureFlags();
    this.cnsClient = new CnsApiClient(config.cnsApiUrl);
    this.sendGridClient = new SendGridClient(config.sendGridApiKey);
    this.twilioClient = new TwilioClient(config.twilioCredentials);
  }
  
  async sendEmail(recipient, template, data, priority) {
    if (await this.featureFlags.isEnabled('use-cns-api')) {
      // New path: Route through CNS
      return await this.cnsClient.sendNotification({
        recipient,
        channel: 'email',
        template,
        data,
        priority
      });
    } else {
      // Old path: Call SendGrid directly
      return await this.sendGridClient.send({
        to: recipient.email,
        template: template.id,
        data
      });
    }
  }
}
```

**Alternatives Considered**

*Big Bang Migration*
- All services update in one weekend
- **Rejected because:** High risk; no rollback capability; likely to miss edge cases

*Service-by-Service without Wrapper*
- Each service updates code to call CNS API
- **Rejected because:** No rollback capability (requires code change + redeploy); higher risk per service

---

## ADR-009: Same Provider Selection and Template Engine as Proposal 1

**Status:** Accepted

**Context**

Both proposals use the same external providers (SendGrid, Twilio, Azure NH per constraints DC-004, DC-005) and require template rendering.

**Decision**

Use the same provider integrations and template engine (Handlebars.js) as Proposal 1.

**Rationale**

- Providers mandated by existing contracts
- Handlebars.js provides best balance of power and security
- No reason to differ from Proposal 1 on these decisions

See Proposal 1 ADRs for detailed rationale:
- ADR-009: Handlebars.js for Template Rendering
- ADR-010: Webhook Callbacks for Status Updates

---

## ADR-010: Multi-Region Deployment for GDPR Compliance

**Status:** Accepted

**Context**

GDPR requires EU customer data stay in EU region. Same requirement as Proposal 1.

**Decision**

Use the same multi-region deployment strategy as Proposal 1: Active-active US/EU with region-specific routing.

See Proposal 1 ADR-007 for detailed rationale.

---

## Summary

These 10 Architecture Decision Records capture the key technical decisions for the API Gateway with Smart Routing proposal:

1. **Request-Response Pattern**: Chose queues over events for operational simplicity
2. **Service Bus Standard**: Chose Standard over Premium for cost savings
3. **Redis for Caching**: Aggressive caching saves $1,500/month net
4. **Smart Router**: Intelligent batching, rate limiting, circuit breaking
5. **Priority-Based Queues**: Four separate queues for SLA enforcement
6. **Reduced Cosmos DB RU**: 30% reduction enabled by Redis caching
7. **Redis Pub/Sub for Sync API**: Low-latency status updates for sync requests
8. **Wrapper Library**: Feature-flag driven migration for zero-downtime rollback
9. **Same Providers/Templates**: Handlebars.js and provider selection matches Proposal 1
10. **Multi-Region Deployment**: Active-active US/EU for GDPR compliance

**Key Differences from Proposal 1:**
- Request-response vs event-driven (simpler, faster, lower cost)
- Redis caching vs in-memory caching (shared state, atomic operations)
- Smart Router vs independent channel processors (batching, rate limiting coordination)
- Service Bus Standard vs Premium (cost optimization)
- Wrapper library migration vs direct migration (safer rollback)

**Trade-offs:**
- **Simpler** but less extensible
- **Faster** but tighter coupling
- **Cheaper** but less flexible
- **Easier to debug** but harder to add new channels
