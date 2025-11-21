# Architecture Decision Records (ADRs) - Proposal 1: Event-Driven Architecture

**Version:** 1.0  
**Date:** November 21, 2025  
**Proposal:** Event-Driven Architecture for Customer Notification Service

---

## ADR-001: Event-Driven Architecture Pattern

**Status:** Accepted

**Context**

The Customer Notification Service must handle high throughput (50M+ notifications/day), support multiple channels (email, SMS, push), and provide reliability guarantees (zero message loss, at-least-once delivery). The architecture needs to scale independently for different channels, isolate failures, and support gradual migration from 15+ existing services.

We evaluated three primary architectural patterns:
1. **Event-driven architecture** using message topics and subscriptions
2. **Request-response architecture** with priority queues and workers
3. **Hybrid architecture** combining synchronous API with asynchronous processing

**Decision**

We will implement an **event-driven architecture** using Azure Service Bus Topics as the primary communication mechanism between components. Notifications flow through a series of events (NotificationSubmitted → NotificationStatus) with specialized processors subscribing to channel-specific filters.

**Rationale**

- **Decoupling**: Components communicate via events, not direct API calls, enabling independent deployment and scaling
- **Scalability**: Each channel processor scales independently based on its subscription queue depth
- **Resilience**: Service Bus message durability guarantees zero message loss; failed components resume from last checkpoint
- **Extensibility**: Adding new channels only requires new subscriptions; no changes to existing components
- **Natural audit trail**: Every event creates a record, simplifying compliance and debugging
- **Proven pattern**: Event-driven architectures are well-established for high-throughput distributed systems

**Consequences**

*Positive:*
- Independent scaling: Email processor can scale to 15 pods while SMS stays at 5 pods
- Fault isolation: Email processor failure doesn't affect SMS or push channels
- Flexible routing: Service Bus topic filters enable sophisticated message routing
- Easy to add new channels or processing stages without modifying existing code
- Built-in retry and dead-letter queue support from Service Bus

*Negative:*
- Increased complexity: Debugging requires tracing through multiple components and events
- Eventual consistency: Status updates lag behind actual sends (mitigated by correlation IDs for sync requests)
- Higher infrastructure costs: Service Bus Premium ($1,340/month) required for throughput
- Operational overhead: More components to monitor, more event flows to understand
- Testing complexity: End-to-end tests require event simulation and asynchronous assertions

**Alternatives Considered**

*Request-Response with Queues*
- Simpler to understand and debug (traditional queue-worker pattern)
- Lower infrastructure costs (Standard Service Bus sufficient)
- **Rejected because:** Tighter coupling between components; all channels share worker pool (can't scale independently); harder to add new channels without code changes

*Hybrid Architecture*
- Best of both worlds: synchronous API with asynchronous backend processing
- **Rejected because:** Adds complexity without significant benefits over pure event-driven; still requires event handling for status updates

---

## ADR-002: Azure Service Bus Topics vs Queues

**Status:** Accepted

**Context**

Azure Service Bus offers two message routing patterns:
1. **Queues**: Point-to-point, single consumer per message
2. **Topics with subscriptions**: Publish-subscribe, multiple consumers per message

The notification service needs to route each notification to potentially multiple channel processors based on the requested channel(s). For example, a notification with `channel: "all"` should go to email, SMS, and push processors.

**Decision**

Use **Azure Service Bus Topics** with channel-specific subscription filters instead of separate queues per channel.

**Rationale**

- **Fan-out capability**: Single notification event can be delivered to multiple processors (multi-channel notifications)
- **Flexible routing**: SQL-like filter expressions enable sophisticated routing logic:
  - `channel='email' OR channel='all'` routes to email processor
  - `priority='critical'` could route to high-priority processor
- **Centralized event publishing**: API Gateway publishes once; Service Bus handles distribution
- **Consistent event ordering**: All processors see events in same order (within partition)
- **Simplified API layer**: No logic to determine which queues to publish to; just publish event and let filters handle routing

**Consequences**

*Positive:*
- Multi-channel notifications work naturally (one publish, multiple subscriptions receive)
- Adding new channels only requires creating new subscription with filter
- Can implement sophisticated routing (priority-based, region-based, A/B testing)
- Reduced API Gateway complexity (no routing logic)

*Negative:*
- Topics are more expensive than queues (Premium tier required for high throughput)
- Slightly higher latency than direct queue access (topic → subscription overhead ~5-10ms)
- Filter evaluation adds CPU overhead on Service Bus
- More complex to reason about message flow (implicit routing via filters)

**Alternatives Considered**

*Separate Queue per Channel*
- Simpler model: One queue for email, one for SMS, one for push
- **Rejected because:** API Gateway must determine which queue(s) to publish to; multi-channel notifications require multiple publish operations; changes to routing logic require API Gateway updates

*Single Queue with Workers Routing*
- Workers pull from single queue and route to appropriate channel handler
- **Rejected because:** Doesn't provide isolation between channels; all workers must understand all channels; harder to scale channels independently

---

## ADR-003: Synchronous vs Asynchronous API Patterns

**Status:** Accepted

**Context**

Consuming services have different latency requirements:
- **Critical notifications** (security alerts, account lockouts): Must know immediately if delivery failed (synchronous)
- **Transactional notifications** (order confirmations): Prefer confirmation but can tolerate async (flexible)
- **Marketing notifications**: Fire-and-forget is acceptable (asynchronous)

We need to decide whether to support:
1. Only asynchronous API (always return tracking ID immediately)
2. Only synchronous API (always wait for delivery confirmation)
3. Both patterns with caller-controlled option

**Decision**

Support **both synchronous and asynchronous API patterns** with caller-controlled behavior via `options.async` flag (default: `true` for async).

**Rationale**

- **Flexibility**: Different use cases have different latency vs reliability tradeoffs
- **Migration friendly**: Allows consuming services to choose pattern that matches their existing behavior
- **Default to async**: Most notifications (95%) are non-critical and benefit from async pattern (lower latency, higher throughput)
- **Opt-in sync**: Critical notifications can opt into sync pattern by setting `options.async: false`
- **Future-proof**: Can add more sophisticated options later (e.g., "wait for send confirmation but not delivery")

**Consequences**

*Positive:*
- Consuming services with critical notifications get immediate feedback on failures
- Non-critical notifications enjoy low latency (P95 < 200ms async vs P95 < 2s sync)
- API Gateway doesn't hold connections open for non-critical requests (better throughput)
- Flexible migration path (services can start async, move to sync for critical only)

*Negative:*
- Increased API Gateway complexity (must handle correlation, timeout, event subscription for sync requests)
- Synchronous requests hold API Gateway threads/connections (limits max concurrent sync requests)
- Risk of timeout issues if status events delayed (sync requests abort after 2s)
- Operational complexity (must monitor both async and sync metrics separately)

**Implementation Details**

*Asynchronous Flow:*
1. API Gateway publishes event to Service Bus
2. Immediately returns 202 Accepted with tracking ID
3. Caller polls GET /notifications/{trackingId} for status

*Synchronous Flow:*
1. API Gateway publishes event with correlation ID
2. Subscribes to status topic with correlation ID filter
3. Waits up to 2 seconds for status event
4. Returns 200 OK with delivery status or 504 Gateway Timeout

**Alternatives Considered**

*Async Only*
- Simpler implementation, no correlation handling
- **Rejected because:** Critical notifications need immediate feedback; forces all consumers to poll for status

*Sync Only*
- Simpler API contract (always returns delivery status)
- **Rejected because:** Higher latency for all requests; holds connections for non-critical notifications; lower max throughput

---

## ADR-004: Cosmos DB as Primary Data Store

**Status:** Accepted

**Context**

The notification service requires persistent storage for:
- Customer preferences (12M records, 24 GB, read-heavy, low latency)
- Notification tracking (50M/day × 90 days = 4.5B records, 4.5 TB, write-heavy)
- Templates (500 templates, 25 MB, read-only)
- Audit logs (50M/day × 90 days, 4.5 TB, append-only)

Requirements:
- Multi-region deployment for GDPR data residency (EU customers → EU region)
- Low latency reads (P95 < 50ms) and writes (P95 < 100ms)
- Strong consistency for preference updates (immediate effect)
- High availability (99.99%+)
- Automatic scaling to handle traffic spikes

Storage options evaluated:
1. **Azure Cosmos DB** (NoSQL, globally distributed)
2. **Azure SQL Database** (Relational, single-region primary)
3. **Azure Table Storage** (NoSQL, cheaper but limited query capabilities)

**Decision**

Use **Azure Cosmos DB with SQL API** as the primary data store for all persistent data (preferences, tracking, templates, audit logs).

**Rationale**

- **Global distribution**: Multi-region writes enable data residency compliance (EU data stays in EU)
- **Automatic scaling**: Autoscale RU/s from 1,000 to 100,000 based on traffic (handles spikes)
- **Low latency**: Single-digit millisecond reads (P50 < 5ms with Session consistency)
- **Flexible schema**: JSON documents adapt easily to new requirements without migrations
- **Built-in indexing**: Automatic indexing on all fields; custom indexes for optimization
- **Comprehensive SLAs**: 99.999% availability (multi-region), P99 latency guarantees
- **TTL support**: Automatic deletion of old audit logs after 90 days (compliance requirement)

**Consequences**

*Positive:*
- No manual replication setup (multi-region writes built-in)
- GDPR compliance straightforward (partition by region/customerId)
- Elastic scaling handles Black Friday 10x spikes without pre-provisioning
- JSON schema flexibility allows rapid iteration on data models
- Built-in change feed for audit trail and event sourcing
- Automatic failover in region outages (RPO < 5 minutes)

*Negative:*
- **Cost**: Most expensive option (~$5,840/month vs ~$2,000/month for Azure SQL)
- Request Unit (RU) model requires capacity planning and monitoring
- Eventual consistency by default (must opt into Session or Strong for preferences)
- Complex pricing model (RU/s + storage + multi-region replication)
- No JOIN operations (must denormalize data or make multiple queries)
- Hot partition risk (though mitigated by good partition key design)

**Partition Key Design**

- `customer-preferences`: `/customerId` - Perfect distribution, each customer is independent
- `notification-tracking`: `/trackingId` - Each notification is independent, avoids hot partitions
- `templates`: `/templateId` - Small dataset, fully cached in application
- `audit-logs`: `/date` - Time-based partitioning, old partitions auto-deleted with TTL

**Alternatives Considered**

*Azure SQL Database*
- Relational model, ACID transactions, familiar SQL
- **Rejected because:** Single-region primary limits data residency options; manual replication setup complex; higher maintenance overhead (patching, backups); doesn't scale as elastically

*Azure Table Storage*
- Cheap ($0.045 per GB vs $0.25 for Cosmos DB)
- **Rejected because:** Limited query capabilities (only partition key + row key); no multi-region writes; no TTL support; no change feed; P95 latency 50-100ms vs < 10ms for Cosmos DB

*Hybrid (Cosmos DB + Azure SQL)*
- Use Cosmos DB for high-throughput tracking, Azure SQL for preferences
- **Rejected because:** Operational complexity of managing two databases; cross-database queries difficult; duplicates effort (monitoring, backup, security)

---

## ADR-005: Cache-Aside Pattern for Customer Preferences

**Status:** Accepted

**Context**

Customer preferences are read on every notification request to check opt-out status, channel preferences, and quiet hours. At 580 notifications/sec average (2,000/sec peak), this translates to:
- 580 reads/sec = 5,800 RU/s (10 RU per read)
- Cosmos DB cost: ~$3,400/month just for preference reads
- P95 latency: ~50ms per read (adds to API latency)

Caching options:
1. **No caching** (always read from Cosmos DB)
2. **In-memory caching** in API Gateway pods (local cache per pod)
3. **Distributed caching** with Redis (shared cache across all pods)
4. **Cache-aside pattern** (application manages cache, fallback to database)
5. **Read-through cache** (cache layer transparently handles database reads)

**Decision**

Implement **cache-aside pattern with in-memory caching** in API Gateway pods for customer preferences. Cache entries have 5-minute TTL and are invalidated on preference updates.

**Rationale**

- **Cost reduction**: Cache hit rate of 80% reduces Cosmos DB reads by 80% (saves ~$2,700/month on RU consumption)
- **Latency improvement**: In-memory cache read < 1ms vs 50ms database read (improves P95 API latency by ~40ms)
- **Simplicity**: No external dependency (Redis); cache is in-process (faster, lower cost)
- **Acceptable staleness**: 5-minute TTL is acceptable (opt-outs don't need to be instant; business requirement is < 24 hours)
- **Invalidation on write**: Preference updates trigger cache invalidation (ensures consistency for critical changes)

**Consequences**

*Positive:*
- Significant cost savings: ~$2,700/month reduction in Cosmos DB RU costs
- Improved API latency: P95 drops from ~200ms to ~150ms
- Reduced load on Cosmos DB: Prevents hot partition issues on high-volume customers
- No external dependencies: One less component to manage and monitor
- Simple implementation: Standard cache libraries (e.g., node-cache, .NET MemoryCache)

*Negative:*
- **Cache staleness**: Preference updates take up to 5 minutes to propagate across all pods (acceptable per business requirements)
- **Memory overhead**: Each pod caches ~12M preferences × 2 KB = 24 MB (acceptable)
- **Cache warming**: New pods start with cold cache (gradual warm-up over first 5 minutes)
- **No cache statistics**: Can't see cache hit rate across all pods (mitigated with metrics aggregation)
- **Cache invalidation complexity**: Must invalidate cache on all pods after preference update (requires inter-pod communication or eventual consistency)

**Implementation Details**

*Cache Structure:*
```javascript
{
  "cust_123456789": {
    "data": { /* preference object */ },
    "cachedAt": "2025-11-21T10:15:30Z",
    "ttl": 300  // 5 minutes
  }
}
```

*Read Flow:*
1. Check in-memory cache for customerId
2. If found and not expired: Return cached data
3. If not found or expired: Read from Cosmos DB
4. Store in cache with 5-min TTL
5. Return data

*Write Flow (Preference Update):*
1. Write updated preference to Cosmos DB
2. Invalidate cache entry locally (immediate)
3. Publish cache invalidation event to Service Bus (broadcast to other pods)
4. Other pods receive event and invalidate their cache entries

**Alternatives Considered**

*No Caching*
- Simplest implementation, always consistent
- **Rejected because:** High Cosmos DB costs; adds 50ms latency to every request

*Distributed Redis Cache*
- Shared cache across all pods, single source of truth
- **Rejected because:** Additional cost ($1,200/month for Premium Redis); external dependency increases complexity; Redis network call adds ~5ms latency (still faster than Cosmos DB but not as fast as in-memory)

*Read-Through Cache*
- Cache layer (e.g., Azure Cache for Redis) automatically handles database reads
- **Rejected because:** Over-engineered for this use case; higher cost and complexity than in-memory caching

*Longer TTL (e.g., 1 hour)*
- Fewer cache misses, lower database load
- **Rejected because:** Unacceptable staleness for opt-outs (customer expects immediate effect within minutes, not hours)

---

## ADR-006: Exponential Backoff Retry Strategy

**Status:** Accepted

**Context**

Channel processors frequently encounter transient failures:
- Provider rate limits (SendGrid, Twilio, Azure NH)
- Network timeouts or connection resets
- Temporary provider outages or degraded performance
- Database throttling (Cosmos DB 429 responses)

Without retries, transient failures result in permanent delivery failures. With naive retries (fixed interval), we risk overwhelming providers or creating retry storms.

Retry strategies evaluated:
1. **No retry** (fail immediately)
2. **Fixed interval retry** (retry every N seconds)
3. **Exponential backoff** (double delay after each failure)
4. **Exponential backoff with jitter** (add randomness to prevent thundering herd)
5. **Circuit breaker** (stop retrying after N failures, wait before resuming)

**Decision**

Implement **exponential backoff retry strategy** with the following parameters:
- Initial delay: 1 second
- Multiplier: 2x (double delay after each attempt)
- Max attempts: 5
- Max delay: 16 seconds (2^4 = 16)
- Retry sequence: 1s, 2s, 4s, 8s, 16s
- After 5 failures: Move to dead-letter queue for manual investigation

Additionally, implement **circuit breaker** for provider APIs:
- Open circuit after 10 consecutive failures
- Half-open timeout: 30 seconds
- Close circuit after 3 consecutive successes

**Rationale**

- **Handles transient failures**: Most provider failures are temporary (rate limits, brief outages); retries increase success rate from 92% to 99%+
- **Backs off gracefully**: Exponential backoff gives providers time to recover without overwhelming them
- **Bounded retry time**: Max 31 seconds total retry time (1+2+4+8+16) prevents indefinite retry loops
- **Circuit breaker prevents cascading failures**: Stops sending requests to failing provider, allowing recovery
- **Dead-letter queue for manual intervention**: Persistent failures require investigation (invalid contact, provider ban, etc.)

**Consequences**

*Positive:*
- High delivery success rate (99%+ for valid contacts)
- Graceful handling of provider rate limits (backs off automatically)
- Prevents overwhelming providers during outages (exponential backoff + circuit breaker)
- Clear distinction between transient and permanent failures (DLQ after 5 attempts)
- Aligns with provider best practices (SendGrid, Twilio, Azure NH all recommend exponential backoff)

*Negative:*
- Increased end-to-end latency for retried notifications (up to 31 seconds for 5 attempts)
- Messages stay in Service Bus longer (locks held during retry delay)
- Complex to test (must simulate failures and delays)
- Operational complexity (must monitor retry rates, DLQ depth, circuit breaker state)

**Implementation Details**

*Retry Logic (Pseudocode):*
```javascript
async function processNotification(notification) {
  let attempt = 0;
  let delay = 1000; // 1 second
  
  while (attempt < 5) {
    try {
      await sendToProvider(notification);
      await completeMessage(); // Success, remove from queue
      return;
    } catch (error) {
      if (isTransientError(error)) {
        attempt++;
        if (attempt < 5) {
          await sleep(delay);
          delay *= 2; // Exponential backoff
          continue;
        }
      }
      // Permanent error or max retries exceeded
      await deadLetterMessage(error);
      return;
    }
  }
}

function isTransientError(error) {
  return error.statusCode === 429 || // Rate limit
         error.statusCode === 503 || // Service unavailable
         error.statusCode === 504 || // Gateway timeout
         error.code === 'ETIMEDOUT' || // Connection timeout
         error.code === 'ECONNRESET'; // Connection reset
}
```

*Circuit Breaker States:*
- **Closed**: Normal operation, requests flow through
- **Open**: Provider failing, reject requests immediately (return 503 to caller)
- **Half-open**: Test if provider recovered, allow limited requests

**Alternatives Considered**

*No Retry*
- Simplest implementation
- **Rejected because:** Unacceptable delivery success rate (92% vs 99% requirement)

*Fixed Interval Retry*
- Retry every 5 seconds for 5 attempts
- **Rejected because:** Doesn't give providers time to recover (could worsen rate limit situation); all retries hit provider at same interval (thundering herd)

*Jittered Exponential Backoff*
- Add random jitter to delay (e.g., delay = baseDelay * 2^attempt + random(0, 1000ms))
- **Rejected because:** Added complexity without significant benefit (Service Bus message locks already provide jitter by holding messages)

*Infinite Retry*
- Keep retrying forever until success
- **Rejected because:** Some failures are permanent (invalid email, customer opted out); infinite retries waste resources

---

## ADR-007: Multi-Region Deployment for Data Residency

**Status:** Accepted

**Context**

GDPR requires that EU customer data be stored and processed within the EU (constraint DC-012). Our customer base spans multiple regions:
- North America: 5M customers (42%)
- Europe: 4M customers (33%)
- Asia Pacific: 2M customers (17%)
- Other: 1M customers (8%)

Deployment strategies evaluated:
1. **Single region** (US West) with data encryption
2. **Multi-region active-passive** (US primary, EU replica)
3. **Multi-region active-active** (US and EU both handle writes)
4. **Region-specific routing** (EU customers → EU, US customers → US)

**Decision**

Deploy **multi-region active-active** with Cosmos DB multi-region writes enabled in US West and EU West. Use region-specific routing at APIM layer: EU customers route to EU deployment, non-EU customers route to US deployment.

**Rationale**

- **GDPR compliance**: EU customer data never leaves EU region (stored and processed locally)
- **Low latency**: Customers routed to nearest region (reduces cross-region latency)
- **High availability**: Either region can handle traffic if the other fails
- **Automatic failover**: Cosmos DB automatically fails over to healthy region
- **Regulatory flexibility**: Can add more regions (e.g., Asia Pacific) for future compliance requirements

**Consequences**

*Positive:*
- GDPR compliant: EU data residency guaranteed
- Improved latency: EU customers experience ~100ms lower latency (no cross-Atlantic round-trip)
- Disaster recovery: Either region can handle 100% of traffic
- Future-proof: Can add more regions for other regulatory requirements (e.g., China data laws)
- Cosmos DB multi-region writes handle replication automatically

*Negative:*
- **Cost increase**: 2x infrastructure cost (~$22k/month vs ~$11k/month for single region)
- **Operational complexity**: Must monitor and maintain two regional deployments
- **Data consistency challenges**: Multi-region writes require eventual consistency model
- **Deployment coordination**: Changes must be deployed to both regions (blue-green more complex)
- **Conflicting writes**: Rare conflict resolution needed for preference updates

**Implementation Details**

*APIM Routing Logic:*
```javascript
if (customer.region === 'EU') {
  routeTo('https://api-eu.company.com');
} else {
  routeTo('https://api-us.company.com');
}
```

*Cosmos DB Configuration:*
- Preferred regions: `["West US 2", "West Europe"]`
- Multi-region writes: Enabled
- Consistency level: Session (balance between consistency and performance)
- Automatic failover: Enabled with priority order (Region 1 primary, Region 2 secondary)

*Data Partitioning:*
- Customer preferences: Partition key includes region (`/customerId_EU`, `/customerId_US`)
- Ensures EU data stays in EU region even with multi-region writes

**Alternatives Considered**

*Single Region (US West)*
- Simplest and cheapest (~$11k/month)
- **Rejected because:** Violates GDPR data residency requirement; high latency for EU customers

*Multi-Region Active-Passive*
- US primary handles all writes, EU replica for reads only
- **Rejected because:** EU writes still go to US (violates GDPR); higher latency for EU customers; passive region underutilized

*Physical Separation (Separate US and EU Deployments)*
- Completely independent US and EU systems (no data sharing)
- **Rejected because:** Operational nightmare (two systems to maintain); customers who move regions can't access their data; no shared template library

---

## ADR-008: ULID for Tracking IDs

**Status:** Accepted

**Context**

Every notification requires a unique tracking ID for:
- Status queries (GET /notifications/{trackingId})
- Correlation across distributed systems
- Audit trail and compliance
- Webhook callbacks

Tracking ID formats evaluated:
1. **UUID v4** (random, 36 characters: `550e8400-e29b-41d4-a716-446655440000`)
2. **ULID** (time-sortable, 26 characters: `01HZKM8XQFZ9B3G7H8J5K6M7N8`)
3. **Snowflake ID** (64-bit integer, time-sortable: `1234567890123456`)
4. **Sequential integer** (auto-increment: `1, 2, 3, ...`)

**Decision**

Use **ULID (Universally Unique Lexicographically Sortable Identifier)** for tracking IDs.

**Rationale**

- **Time-sortable**: First 10 characters encode timestamp (millisecond precision); tracking IDs naturally sort by creation time
- **Compact**: 26 characters vs 36 for UUID (shorter URLs, less storage)
- **URL-safe**: Uses Crockford's Base32 alphabet (no special characters)
- **Globally unique**: 128-bit identifier with 80 bits of randomness (collision probability negligible)
- **No coordination required**: Can be generated independently in any pod without coordination
- **Debuggability**: Can extract creation timestamp from ID (helps troubleshooting)

**Consequences**

*Positive:*
- Database queries naturally sort by time (efficient for "recent notifications" queries)
- No need for separate `createdAt` index in Cosmos DB (ULID is the index)
- Shorter IDs improve URL readability and reduce storage (26 chars vs 36)
- Can generate offline without database or coordination service
- Timestamp extraction helps debugging ("this notification was created 3 hours ago")

*Negative:*
- Less familiar than UUID (team must learn ULID format)
- Timestamp component leaks information about creation time (minor security concern)
- Requires ULID library (not built into most languages unlike UUID)
- Very slight predictability (timestamp component is sequential; randomness component prevents guessing)

**Format Specification**

```
ULID: 01HZKM8XQFZ9B3G7H8J5K6M7N8
      |----------|-------------|
      Timestamp   Randomness
      (48 bits)   (80 bits)

Components:
- Timestamp: Unix epoch in milliseconds (01HZKM8X = 1732184130000 ms = Nov 21, 2025 10:15:30 GMT)
- Randomness: Cryptographically secure random value (QFZ9B3G7H8J5K6M7N8)

Encoding: Crockford's Base32 (0-9, A-Z excluding I, L, O, U)
Length: 26 characters
```

**Generation Example:**
```javascript
import { ulid } from 'ulid';

const trackingId = ulid(); // "01HZKM8XQFZ9B3G7H8J5K6M7N8"

// Extract timestamp
const timestamp = ulid.decodeTime(trackingId); // 1732184130000 (milliseconds)
const date = new Date(timestamp); // 2025-11-21T10:15:30.000Z
```

**Alternatives Considered**

*UUID v4*
- Most familiar format, built into all languages
- **Rejected because:** Not time-sortable (random); longer (36 chars); wastes storage space

*Snowflake ID*
- Twitter's 64-bit ID scheme, time-sortable
- **Rejected because:** Requires coordination for machine ID component; 64-bit integers awkward in some languages (JavaScript loses precision); not URL-friendly as decimal integer

*Sequential Integer*
- Simplest possible ID, guaranteed unique with database auto-increment
- **Rejected because:** Requires database coordination (performance bottleneck); predictable (security concern); reveals business metrics (ID 10M means 10 million notifications sent)

---

## ADR-009: Handlebars.js for Template Rendering

**Status:** Accepted

**Context**

Notification templates require variable substitution and basic logic:
- Variable interpolation: `{{orderId}}`, `{{customerName}}`
- Conditional sections: `{{#if isPremium}}Premium Benefits{{/if}}`
- Loops: `{{#each items}}Item: {{name}}{{/each}}`
- Date formatting: `{{formatDate deliveryDate "MMM DD, YYYY"}}`

Template engines evaluated:
1. **Handlebars.js** (logic-less, widely adopted)
2. **Mustache** (completely logic-less, simpler than Handlebars)
3. **EJS** (Embedded JavaScript, full JavaScript in templates)
4. **Liquid** (Shopify's template language, Ruby-inspired)
5. **String interpolation** (built-in template literals)

**Decision**

Use **Handlebars.js** for template rendering across all channels (email, SMS, push, in-app).

**Rationale**

- **Balance of power and safety**: Supports conditionals and loops without allowing arbitrary code execution
- **Widely adopted**: Large community, extensive documentation, proven in production
- **Precompilation**: Templates can be precompiled for performance (10x faster rendering)
- **Helper functions**: Custom helpers for formatting (dates, currency, truncation)
- **Language support**: Available in JavaScript, C#, Java, Python (flexibility for polyglot architecture)
- **Security**: Logic-less by default prevents template injection attacks

**Consequences**

*Positive:*
- Fast rendering: Precompiled templates render in < 5ms (meets P95 latency targets)
- Secure: Logic-less design prevents template injection (no arbitrary code execution)
- Flexible: Helpers enable complex formatting without embedding logic in templates
- Cacheable: Precompiled templates can be cached aggressively (1-hour TTL)
- Familiar: Many developers already know Handlebars (low learning curve)

*Negative:*
- More complex than Mustache (supports more features but slightly harder to learn)
- Template debugging can be difficult (errors surface at render time, not compile time)
- Helpers must be registered globally (all channel processors must share helper registry)
- No type safety (typos in variable names fail silently: `{{orderId}}` vs `{{order_id}}`)

**Template Example**

```handlebars
Subject: Your order {{orderId}} has shipped!

Hi {{customerName}},

Great news! Your order has shipped and will arrive by {{formatDate deliveryDate "MMMM DD, YYYY"}}.

{{#if trackingAvailable}}
Track your package: {{trackingUrl}}
{{/if}}

Order Summary:
{{#each items}}
- {{name}}: {{formatCurrency price}}
{{/each}}

Total: {{formatCurrency orderTotal}}

{{#unless isPremium}}
Upgrade to Premium for free shipping: https://example.com/premium
{{/unless}}

Thanks for shopping with us!
```

**Custom Helpers**
```javascript
Handlebars.registerHelper('formatDate', function(date, format) {
  return moment(date).format(format);
});

Handlebars.registerHelper('formatCurrency', function(amount) {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount);
});

Handlebars.registerHelper('truncate', function(text, length) {
  return text.length > length ? text.substring(0, length) + '...' : text;
});
```

**Alternatives Considered**

*Mustache*
- Simpler than Handlebars (fewer features)
- **Rejected because:** Too limited for complex templates (no helpers, no @index in loops); would require workarounds

*EJS (Embedded JavaScript)*
- Full JavaScript in templates (most powerful)
- **Rejected because:** Security risk (template injection attacks); too much logic in templates; hard to review for non-technical users

*Liquid*
- Shopify's template language (widely used in e-commerce)
- **Rejected because:** JavaScript implementation less mature than Handlebars; smaller community; Ruby-centric documentation

*Built-in Template Literals*
- No external dependency, fast
- **Rejected because:** No conditionals or loops; must manually escape HTML; no precompilation; security risk (code injection)

---

## ADR-010: Webhook Callbacks for Status Updates

**Status:** Accepted

**Context**

Consuming services need to know when notifications are delivered or fail:
- **Polling approach**: Services periodically call GET /notifications/{trackingId}
- **Webhook approach**: CNS calls consuming service webhook URL with status updates
- **WebSocket approach**: Bi-directional real-time connection for status updates
- **Server-Sent Events (SSE)**: Server pushes updates over HTTP connection

Requirements:
- Support asynchronous status updates (notification sent hours after submission)
- Low operational overhead for consuming services
- Reliable delivery of status updates (at-least-once guarantee)
- Secure communication (authenticate webhook callbacks)

**Decision**

Implement **webhook callbacks** as the primary mechanism for status updates. Consuming services optionally provide a webhook URL in the notification request. Status Tracker service triggers HTTP POST to webhook URL when status changes.

**Rationale**

- **Push model**: Consuming services don't need to poll (reduces API load by ~90%)
- **Real-time updates**: Status updates delivered immediately when available (< 1 second latency)
- **Standard pattern**: Webhooks are industry standard (SendGrid, Twilio, Stripe all use webhooks)
- **Flexible**: Consuming services can provide different webhook URLs per notification
- **Optional**: Services that don't care about status don't need to implement webhook endpoint

**Consequences**

*Positive:*
- No polling overhead: Saves 1000s of API calls per second (estimated 90% reduction in status query traffic)
- Real-time updates: Consuming services notified within 1 second of status change
- Scales better: Push model scales better than pulling (fewer requests to API)
- Standard integration: Webhook pattern familiar to all engineers

*Negative:*
- **Endpoint complexity**: Consuming services must implement and maintain webhook endpoint
- **Security concerns**: Must verify webhook authenticity (HMAC signature)
- **Delivery reliability**: Webhooks can fail (network issues, service downtime); requires retry logic
- **No guaranteed ordering**: Status updates may arrive out of order (sent before delivered)
- **Testing complexity**: Webhook endpoints harder to test than API polling (requires public URL or tunneling)

**Implementation Details**

*Webhook Payload:*
```json
POST https://service.company.com/webhook/notifications
Content-Type: application/json
X-CNS-Signature: sha256=abc123def456... (HMAC-SHA256 signature)
X-CNS-Timestamp: 2025-11-21T10:16:00Z

{
  "trackingId": "01HZKM8XQFZ9B3G7H8J5K6M7N8",
  "status": "delivered",
  "channel": "email",
  "timestamp": "2025-11-21T10:16:00Z",
  "provider": "sendgrid",
  "providerMessageId": "msg_sg_xyz123",
  "metadata": {
    "correlationId": "req-abc-123"
  }
}
```

*Signature Verification:*
```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(JSON.stringify(payload));
  const expectedSignature = `sha256=${hmac.digest('hex')}`;
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}
```

*Retry Logic:*
- Attempt 1: Immediate
- Attempt 2: After 5 seconds
- Attempt 3: After 30 seconds
- Attempt 4: After 5 minutes
- After 4 failures: Log error and move on (consuming service must poll for status)

*Timeout:*
- Webhook calls timeout after 5 seconds (prevents hanging on slow/down services)

**Security**

- **HMAC signature**: Included in X-CNS-Signature header, consuming services verify with shared secret
- **Timestamp validation**: X-CNS-Timestamp header prevents replay attacks (reject if > 5 minutes old)
- **HTTPS only**: Webhook URLs must use HTTPS (TLS encryption)
- **IP allowlisting**: Consuming services can allowlist CNS outbound IPs

**Alternatives Considered**

*Polling Only*
- Consuming services call GET /notifications/{trackingId} periodically
- **Rejected because:** High API load (1000s of requests/sec); high latency (minutes until status discovered); wasted requests (most polls return "no change")

*WebSocket*
- Bi-directional real-time connection for status updates
- **Rejected because:** Complex to implement and maintain; requires persistent connections (scaling challenges); overkill for infrequent status updates

*Server-Sent Events (SSE)*
- Server pushes updates over HTTP connection (simpler than WebSocket)
- **Rejected because:** Requires consuming services to keep connection open (scaling challenges); not standard pattern for service-to-service communication

*Message Queue*
- Publish status updates to consuming service's Azure Service Bus queue
- **Rejected because:** Requires consuming services to have Service Bus; tighter coupling; more complex integration

---

## Summary

These 10 Architecture Decision Records capture the key technical decisions for the Event-Driven Architecture proposal:

1. **Event-Driven Architecture Pattern**: Chose events over request-response for decoupling and scalability
2. **Azure Service Bus Topics**: Chose topics over queues for fan-out and flexible routing
3. **Synchronous + Asynchronous API**: Support both patterns for flexibility
4. **Cosmos DB**: Chose Cosmos DB over SQL for global distribution and low latency
5. **Cache-Aside Pattern**: In-memory caching reduces costs and latency
6. **Exponential Backoff Retry**: Graceful failure handling with circuit breakers
7. **Multi-Region Deployment**: Active-active regions for GDPR compliance
8. **ULID for Tracking IDs**: Time-sortable unique identifiers
9. **Handlebars.js Templates**: Balance of power and security for rendering
10. **Webhook Callbacks**: Push model for real-time status updates

Each decision balances trade-offs between complexity, cost, performance, and reliability to meet the system's requirements.
