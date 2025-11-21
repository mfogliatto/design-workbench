# Critical Analysis: Proposal 1 - Event-Driven Architecture

## Executive Summary

This critique examines the event-driven architecture proposal for the Customer Notification Service, with emphasis on **scalability concerns** and **cost efficiency**. While the proposal demonstrates architectural sophistication and excellent theoretical scalability, it exhibits critical flaws in cost modeling, hidden scaling bottlenecks, and unrealistic assumptions about operational overhead.

**Risk Rating: MEDIUM-HIGH** ⚠️

**Key Findings**:
- ❌ **Cost projections underestimate reality by 30-40%** - missing Service Bus message duplication costs, Cosmos DB write amplification, and true operational overhead
- ❌ **Service Bus Premium required but under-specified** - 2 messaging units insufficient for peak load; need 4-6 units ($2,680-4,020/month vs $1,340 budgeted)
- ⚠️ **Cosmos DB write amplification not accounted for** - event-driven pattern generates 3-5x more writes than acknowledged (status tracking, event logs, webhook records)
- ⚠️ **Event fan-out creates message explosion** - single notification becomes 3-5 Service Bus messages; actual throughput 10,000-15,000 msg/sec not 2,000
- ⚠️ **Synchronous API pattern fundamentally broken** - waiting for events across distributed system violates timeout constraints and creates connection exhaustion
- ✅ **Independent channel scaling is legitimate advantage**
- ✅ **Audit trail claim is valid**

**Bottom Line**: This architecture can scale to requirements but at **40% higher cost than projected** ($130k/month vs $93k claimed). The synchronous API pattern should be eliminated entirely. Event-driven complexity is justifiable only if organization already has deep expertise in this pattern.

---

## Methodology

### Analysis Approach
This critique employs:
1. **Detailed cost modeling** with Azure pricing calculator validation
2. **Throughput calculations** accounting for message duplication and fan-out
3. **Failure scenario simulation** based on production incident patterns
4. **Comparative analysis** against stated requirements and constraints
5. **Resource sizing validation** using Azure Service Bus and Cosmos DB calculators

### Key Assumptions Validated
- Peak throughput: 2,000 notifications/sec (stated requirement)
- Message size: ~2KB per notification payload
- Event fan-out factor: 3-5 messages per notification (calculated)
- Multi-region deployment: US West + EU West for GDPR compliance
- 90-day audit retention requirement

### Source Documents Referenced
- [Requirements](../../sources/requirements.md) - NFR-001 through NFR-027
- [Facts](../../sources/facts.md) - Scale data and provider SLAs
- [Overview](../../sources/overview.md) - Problem statement and scope

---

## Critical Design Flaws

### 1. Service Bus Capacity Severely Underestimated

**Flaw**: Proposal budgets for Service Bus Premium with 2 messaging units ($1,340/month) claiming capacity for 2,000 msg/sec throughput. This is **fundamentally incorrect**.

**Reality Check**:
```
Event fan-out per notification:
- 1x NotificationSubmitted event → notifications-submitted topic
- 1x channel-specific subscription delivery (email, SMS, or push)
- 1x NotificationSent event → notifications-status topic  
- 1x status tracker subscription delivery
- Optional: correlation event for sync API (15-20% of requests)

Actual Service Bus throughput required:
- Normal flow: 2,000 notifications/sec × 4 messages = 8,000 msg/sec
- With sync API overhead: 8,000 + (2,000 × 0.18) = 8,360 msg/sec
- Peak burst safety margin (2x): 16,720 msg/sec

Azure Service Bus Premium capacity:
- Per messaging unit: ~3,000 msg/sec sustained
- 2 messaging units: 6,000 msg/sec (INSUFFICIENT)
- Required: 6 messaging units: 18,000 msg/sec ($4,020/month)
```

**Cost Impact**: +$2,680/month ($4,020 actual vs $1,340 budgeted) = **+200% Service Bus cost overrun**

**Scaling Bottleneck**: At 150M/day (3x growth), event fan-out creates 25,000+ msg/sec peak load requiring 9-10 messaging units ($6,030-6,700/month). This scales poorly compared to queue-based approaches.

### 2. Cosmos DB Write Amplification Not Accounted For

**Flaw**: Proposal estimates 10,000 RU/s for Cosmos DB based on simple read/write calculations. Event-driven architecture generates far more database writes than acknowledged.

**Hidden Write Operations**:
```
Per notification, Cosmos DB writes include:
1. Preference lookup (cached, but cache misses: ~20% = 116/sec writes from cache refreshes)
2. Initial tracking record creation (580/sec)
3. Status update from NotificationSent event (580/sec)
4. Webhook delivery record (if configured, ~30% = 174/sec)
5. Audit log entry for compliance (580/sec)
6. Event correlation records for sync API (104/sec)

Total write operations:
- 580 + 580 + 174 + 580 + 104 + 116 = 2,134 writes/sec (not 580)
- At 10 RU per write: 21,340 RU/sec (not 2,900 estimated)

Actual Cosmos DB provisioning required:
- Writes: 21,340 RU/sec
- Reads: 5,800 RU/sec (preference + status queries)
- Total: 27,140 RU/sec → provision 30,000 RU/sec
- Cost: $17,520/month (vs $5,840 budgeted)
```

**Cost Impact**: +$11,680/month = **+200% Cosmos DB cost overrun**

**Scaling Impact**: At 150M/day (3x), write amplification becomes catastrophic: 90,000 RU/sec required ($52,560/month vs $18k projected).

### 3. Synchronous API Pattern is Architecturally Broken

**Flaw**: Proposal claims to support synchronous API calls where caller waits for delivery confirmation. This is **fundamentally incompatible** with event-driven architecture.

**Why This Fails**:
```
Synchronous request flow:
1. API Gateway receives request (0ms)
2. Publishes event to Service Bus (50-100ms)
3. Event propagates to channel processor (100-200ms)
4. Processor calls provider (SendGrid: 5s, Twilio: 3s, Azure NH: 2s)
5. Status event published (50-100ms)
6. Status Tracker receives event (100-200ms)
7. API Gateway receives correlation event (50-100ms)

Total latency: 5.3-8.6 seconds (email worst case)

Problems:
- NFR-003 requires P95 < 2 seconds for sync requests (VIOLATED by 4-6 seconds)
- API Gateway holds open HTTP connections for 5-8 seconds (connection pool exhaustion)
- At 200 sync requests/sec, need 1,000-1,600 concurrent connections
- Event correlation complexity creates race conditions
- Timeout handling requires compensating transactions
```

**Architectural Contradiction**: Event-driven patterns optimize for decoupling and eventual consistency. Synchronous request-response requires tight coupling and immediate consistency. **These are opposing goals**.

**Recommendation**: **Eliminate synchronous API pattern entirely**. Force all callers to use async pattern with webhook callbacks. This is the only honest event-driven approach.

### 4. Event Storm Risks Underestimated

**Flaw**: Proposal mentions "event storms" as a risk but doesn't quantify the failure scenario or impact.

**Realistic Failure Scenario**:
```
Trigger: SendGrid experiences 30-second outage at 9:00 AM (peak traffic)

Timeline:
T+0s: SendGrid returns 503 errors
T+5s: 1,000 notifications queued in email processor subscription
T+10s: Processors begin exponential backoff retries
T+15s: Circuit breaker triggers (10 consecutive failures)
T+30s: SendGrid recovers

BUT: During circuit breaker open period:
- Email processor stops pulling from subscription
- Subscription accumulates 60,000 messages (30s × 2,000/sec)
- Service Bus auto-scaling adds messaging capacity
- When circuit breaker closes, 60,000 messages rush through
- Status events flood notifications-status topic
- Status Tracker overwhelmed (3-5 minute backlog)
- API Gateway correlation timeouts cascade
- Webhook callbacks flood consuming services

Event storm amplification:
- 60,000 queued notifications
- × 2 events each (sent + status)
- = 120,000 event burst over 2-3 minutes
- Overwhelms Status Tracker (single-threaded correlation logic)
- Cosmos DB write throttling begins (hot partition on trackingId)
```

**Impact**: 5-10 minute recovery time (violates 99.9% availability SLA). Potential message loss if Status Tracker crashes before updating Cosmos DB.

**Mitigation Required**: 
- Separate status tracking per channel (not single Status Tracker)
- Cosmos DB rate limiting with queue backpressure
- Maximum retry queue depth limits
- Graduated circuit breaker recovery (not binary open/close)

### 5. Cosmos DB Partitioning Strategy Will Create Hot Partitions

**Flaw**: Proposal partitions preferences by `customerId` which is correct, but partitions tracking data by `trackingId` which **will create hot partitions**.

**Problem Analysis**:
```
Tracking data access pattern:
- Write: 580/sec uniformly distributed across trackingIds ✅
- Read: Status queries concentrated on RECENT trackingIds
  - API queries for "my last 10 notifications" hit same time window
  - Webhook retry queries hit same recent IDs
  - Support dashboard queries focused on last 24 hours

Result: 80% of read traffic targets 5% of partition key space (recent time)

Hot partition impact:
- Partition can handle ~10,000 RU/sec before throttling
- Recent time window receives 4,640 RU/sec (80% of 5,800 reads)
- Leaves only 5,360 RU/sec for writes
- BUT writes to this partition: 464 RU/sec (80% of 580)
- Total: 5,104 RU/sec (within limit)

Actually... this might be OK for 50M/day, BUT:
At 150M/day (3x growth):
- Recent partition receives 13,920 RU/sec reads + 1,392 writes = 15,312 RU/sec
- EXCEEDS single partition limit (10,000 RU/sec)
- Results in 429 throttling errors
```

**Recommendation**: Partition tracking data by `customerId` + `timestamp_bucket` (hourly or daily buckets) to spread reads across partitions. Accept cross-partition queries for status lookups.

---

## Scale-Specific Failure Analysis

### Scenario 1: Black Friday Traffic Spike (10x Normal Volume)

**Trigger**: Promotional campaign generates 20,000 notifications/sec for 2 hours.

**System Behavior**:
```
Service Bus:
- Message ingestion: 20,000 × 4 events = 80,000 msg/sec
- 6 messaging units support 18,000 msg/sec sustained
- Service Bus throttles at ingestion (HTTP 429 errors)
- API Gateway retry logic exacerbates load
- Messages queued in Service Bus (max 80GB capacity)
- Queue depth: 80,000 msg/sec × 120 seconds = 9.6M messages
- At 2KB/msg: 19.2GB queued (within capacity)

Channel Processors:
- Email processor auto-scales to 50 pods (from 10)
- Each pod processes 60 msg/sec = 3,000 msg/sec total
- Backlog accumulation: (20,000 - 3,000) × 7,200s = 122M messages queued
- SendGrid rate limit: 10,000 emails/sec (sufficient)
- Processing backlog clears in 11 hours AFTER spike ends

Cosmos DB:
- Write load spikes to 214,000 RU/sec (10x normal)
- Provisioned capacity: 30,000 RU/sec (if flaw #2 addressed)
- Massive throttling (85% of writes rejected)
- Auto-scaling takes 5-10 minutes to reach 214,000 RU/sec
- Cost during spike: $125/hour for duration of spike
```

**Impact**: 
- 11-hour backlog processing (violates SLA)
- Cosmos DB auto-scaling cost: $250-500 for 2-hour spike
- Some notifications delayed by 13+ hours
- Critical notifications mixed with promotional backlog

**Verdict**: **FAILURE** - System cannot handle 10x spike without massive SLA violations.

**Mitigation**:
- Separate Service Bus topics for critical vs promotional notifications
- Pre-scale Cosmos DB before known events
- Reject promotional traffic during overload (admission control)

### Scenario 2: Cascading Failure from Cosmos DB Regional Outage

**Trigger**: Azure Cosmos DB EU West region experiences 30-minute outage (per Azure SLA, allowed).

**System Behavior**:
```
T+0: EU West Cosmos DB unreachable
- Multi-region failover to US West (automatic, 2-3 minute window)
- During failover: all database operations fail
- API Gateway: preference lookups fail → reject requests OR allow without validation
- If allow: privacy violations (send to opted-out customers)
- If reject: lose messages (violates at-least-once guarantee)

T+3min: Failover complete to US West
- EU customer data readable from US West (but stale by 2-3 minutes)
- EU data residency violation (GDPR) - customer data accessed from US region
- Writes go to US West, replicate back to EU West when recovered

T+30min: EU West recovers
- Replication backlog: 30 minutes of writes (1.05M write operations)
- Conflict resolution for concurrent updates (preference changes)
- Some preference updates lost if conflicts resolved incorrectly
```

**GDPR Compliance Impact**: During failover, EU customer data accessed from US region violates GDPR data residency requirements. **Critical compliance violation**.

**Verdict**: **FAILURE** - Multi-region Cosmos DB strategy violates mandatory constraint DC-012 (EU data must stay in EU).

**Required Fix**: Separate Cosmos DB instances per region (not multi-region replication). Increases cost by 100% (need two full Cosmos DB instances) but maintains compliance.

### Scenario 3: SendGrid Rate Limit Breach During Email Campaign

**Trigger**: Marketing team launches email campaign generating 12,000 emails/sec (exceeds SendGrid's 10,000/sec limit).

**System Behavior**:
```
T+0: Email processor submits 12,000 emails/sec to SendGrid
- SendGrid returns 429 rate limit errors for 2,000/sec
- Processor interprets as transient failure
- Exponential backoff retry: attempt 1s, 2s, 4s, 8s, 16s later
- Retry storm: original 2,000/sec + 2,000 retries = 4,000/sec over limit

T+10s: Circuit breaker triggers (10 consecutive 429s)
- Email processor stops sending for 30 seconds
- 360,000 emails queued (12,000 × 30)

T+40s: Circuit breaker half-open, test request succeeds
- Circuit closes, 360,000 emails rush through
- 6,000/sec sustained (within SendGrid limit)
- Backlog clears in 60 seconds

T+100s: Campaign continues, same pattern repeats every 90 seconds
```

**Issue**: Proposal claims "rate limiting to respect provider limits" but doesn't specify mechanism. Token bucket? Leaky bucket? Per-processor or global?

**If per-processor rate limiting**: Each email processor pod has local token bucket. 10 pods × 1,000/sec = 10,000/sec. BUT during scale-out to 50 pods, global rate becomes 50,000/sec (5x over limit).

**Verdict**: ⚠️ **INCOMPLETE DESIGN** - Rate limiting mechanism not specified. Local rate limiting breaks during auto-scaling. Requires global distributed rate limiter (adds latency and complexity).

---

## Cost Analysis: Reality vs. Projection

### Proposal's Cost Estimate (50M/day)
```
Service Bus Premium: $1,340/month (2 messaging units)
Cosmos DB: $5,840/month (10,000 RU/s)
AKS: $600/month
APIM: $2,800/month
Monitoring: $400/month
Other: $50/month
Infrastructure Total: $11,030/month

Providers: $82,115/month
Grand Total: $93,145/month
```

### Corrected Cost Estimate (50M/day)

**Service Bus Premium**: 
- Required: 6 messaging units for 18,000 msg/sec capacity
- Cost: $4,020/month (+$2,680)

**Cosmos DB**:
- Corrected RU/s: 30,000 (accounting for write amplification)
- Multi-region: US West (30,000 RU) + EU West (30,000 RU) = 60,000 RU total
- Cost: $35,040/month (vs $5,840 budgeted) (+$29,200)
- *Note: Separate regions required for GDPR compliance, not multi-region replication*

**AKS**:
- API Gateway: 5 pods × $80 = $400/month
- Email Processor: 10-25 pods × $80 = $800-2,000/month (avg $1,400)
- SMS Processor: 5-10 pods × $80 = $400-800/month (avg $600)
- Push Processor: 3-8 pods × $80 = $240-640/month (avg $440)
- Status Tracker: 5-10 pods × $80 = $400-800/month (avg $600)
- Total: $3,440/month (vs $600 budgeted) (+$2,840)

**Application Insights**:
- Event telemetry: 8,000-16,000 events/sec × 86,400 sec/day = 691M-1.38B events/day
- At $2.30/GB, ~500GB/month telemetry
- Cost: $1,150/month (vs $400 budgeted for all monitoring) (+$750)

**Corrected Infrastructure Total**: $46,450/month (vs $11,030 budgeted)
**Corrected Grand Total**: $128,565/month (vs $93,145 budgeted)

**Cost Overrun: +$35,420/month (+38%)**

### Corrected Cost Estimate (150M/day - 3x Growth)

**Service Bus Premium**:
- 10 messaging units: $6,700/month

**Cosmos DB**:
- US West: 90,000 RU/s = $52,560/month
- EU West: 90,000 RU/s = $52,560/month
- Total: $105,120/month

**AKS**: 
- 3x scale: ~$10,320/month

**Application Insights**:
- 3x telemetry: $3,450/month

**Infrastructure Total**: $128,390/month
**Providers**: ~$246,000/month
**Grand Total**: $374,390/month (vs $264k projected = +42% overrun)

**Cost per 1000 notifications**: $2.50 (vs $1.76 projected = +42%)

---

## Security and Compliance Vulnerabilities

### 1. GDPR Data Residency Violation

**Vulnerability**: Multi-region Cosmos DB replication violates DC-012 (EU data must stay in EU).

**Scenario**: EU customer's preference data stored in EU West replicates to US West. During failover, US region serves EU customer queries. **GDPR violation**.

**Fix**: Separate Cosmos DB accounts per region. Route EU customers to EU Cosmos DB exclusively. Doubles Cosmos DB costs but maintains compliance.

**Severity**: CRITICAL ❌ - This is a mandatory design constraint violation that could result in regulatory fines.

### 2. Event Log Contains PII Without Encryption

**Vulnerability**: Service Bus messages contain customer email addresses and phone numbers (PII) but proposal doesn't specify encryption at rest for Service Bus.

**Azure Service Bus Premium**: Supports customer-managed keys for encryption at rest, but must be explicitly configured.

**Missing from proposal**:
- Customer-managed key (CMK) configuration
- Key rotation policy
- Access control for key vault

**Severity**: HIGH ⚠️ - NFR-015 requires PII encryption at rest.

### 3. Status Tracker Has Unrestricted Cosmos DB Access

**Vulnerability**: Status Tracker service updates all tracking records with full write access to Cosmos DB. Compromised Status Tracker could modify or delete any notification history.

**Principle of Least Privilege Violation**: Status Tracker should have write access only to tracking records it creates, not global write access.

**Recommendation**: Implement Cosmos DB RBAC with scoped access per service. Status Tracker gets write access to specific container only.

**Severity**: MEDIUM ⚠️

---

## Operational Complexity Assessment

### Monitoring Explosion

**Problem**: Event-driven architecture generates 10-15x more metrics than request-response systems.

**Metrics Required per Component**:
```
API Gateway: 12 metrics (request rate, latency, error rate, etc.)
Service Bus Topics (2): 20 metrics each (queue depth, throughput, throttling, DLQ depth)
Channel Processors (3): 15 metrics each (processing rate, provider latency, retry count)
Status Tracker: 15 metrics (correlation success rate, webhook delivery rate)
Cosmos DB: 30 metrics (RU consumption, throttling, latency per operation type)

Total: 12 + 40 + 45 + 15 + 30 = 142 metrics to monitor

Alert rules required: 40-50 (based on best practices)
Dashboards: 8-10 (per-service + overview)
```

**Operational Burden**: Requires dedicated SRE engineer just for monitoring configuration and alert tuning. First 3 months will be high-alert-noise period (alert fatigue).

**Cost Impact**: 1 FTE SRE × $180k/year = $15k/month (not included in cost estimates).

### Debugging Complexity

**Scenario**: Customer reports "I didn't receive my password reset email."

**Investigation Steps**:
```
1. Query Cosmos DB for tracking ID (if customer provides it) OR
2. Query by customer ID + timestamp window (cross-partition query, expensive)
3. Find notification record, check status
4. If "sent" status: Query Service Bus DLQ for NotificationSubmitted event
5. Check Application Insights for event correlation ID
6. Trace event flow across 4 services:
   - API Gateway → Service Bus Topic → Email Processor → Status Tracker
7. Check each component's logs for errors
8. Check SendGrid delivery logs (external system)
9. Correlate timestamps across 5 systems with clock skew

Time to resolution: 20-45 minutes for experienced engineer
Time to resolution: 2-4 hours for new team member
```

**Compare to request-response architecture**: Direct log query by tracking ID, 2-5 minutes to resolution.

**Operational Impact**: Increased MTTR (Mean Time To Resolution) = lower effective availability.

### Deployment Complexity

**Challenge**: Event schema evolution requires coordinated deployment of multiple services.

**Example**: Add new field to NotificationSubmitted event (`priority_level`).

**Deployment sequence**:
```
1. Update event schema definition
2. Deploy Status Tracker (must accept both old + new schema)
3. Deploy Channel Processors (must accept both old + new schema)
4. Deploy API Gateway (starts sending new schema)
5. Verify end-to-end flow with new field
6. After 2 weeks (grace period), remove old schema support from processors
```

**Risk**: If deployment order wrong, events rejected → messages sent to DLQ → manual recovery required.

**Operational Impact**: Requires strict deployment runbooks and coordination. Zero-tolerance for mistakes.

---

## Alternative Architecture Recommendations

### Recommendation 1: Hybrid Approach - Queues with Event Sourcing

**Concept**: Use Service Bus Queues (not Topics) for primary workflow, but emit events to separate event store for audit trail.

**Benefits**:
- Simpler message flow (queues not topics) = lower Service Bus costs
- Event sourcing provides audit trail without event-driven complexity
- Retains some loose coupling benefits

**Trade-off**: Slightly tighter coupling than pure event-driven, but 40% cost reduction.

### Recommendation 2: Eliminate Event-Driven Pattern Entirely

**Concept**: Use Proposal 2's queue-worker pattern instead.

**Why**: 
- Organization doesn't have stated event-driven expertise
- Cost savings: $130k vs $90k/month (31% cheaper)
- Faster development: 4 months vs 5 months
- Lower operational burden

**When Event-Driven Makes Sense**: If organization already operates 5+ event-driven services and has dedicated platform team supporting Service Bus Topics at scale.

---

## Risk Assessment and Recommendations

### Critical Risks (Must Fix Before Implementation)

| Risk | Severity | Probability | Mitigation |
|------|----------|-------------|------------|
| GDPR data residency violation | CRITICAL | 100% | Separate Cosmos DB per region (not multi-region) |
| Cost overrun (38%) | HIGH | 90% | Revise cost estimates; secure additional budget |
| Cosmos DB hot partitions at scale | HIGH | 70% | Repartition tracking data by customerId + time bucket |
| Service Bus capacity insufficient | HIGH | 85% | Provision 6 messaging units minimum (not 2) |
| Sync API pattern broken | CRITICAL | 100% | Eliminate synchronous API; force async + webhooks |

### High Risks (Require Mitigation)

| Risk | Severity | Mitigation |
|------|----------|------------|
| Event storms during provider outages | HIGH | Graduated circuit breakers, retry queue depth limits |
| Rate limiting during auto-scale | HIGH | Implement distributed rate limiter with Redis |
| Operational complexity | MEDIUM | Hire dedicated SRE; invest in observability platform |
| Deployment coordination | MEDIUM | Automated deployment pipelines with schema validation |

### Recommendations for Risk Mitigation

1. **Conduct Proof of Concept**: Before full implementation, build spike solution testing:
   - Service Bus throughput at 20,000 msg/sec
   - Cosmos DB write amplification patterns
   - Event correlation for sync API (then likely abandon this pattern)
   - Cost validation over 2-week period

2. **Revise Cost Budget**: Increase infrastructure budget to $130k/month (not $93k). Secure executive approval for 38% increase.

3. **Eliminate Synchronous API**: Force all consumers to use async + webhook pattern. Event-driven and sync request-response are architecturally incompatible.

4. **Hire SRE Expertise**: This architecture requires dedicated operational expertise. Plan for 1-2 FTE SREs.

5. **Consider Alternative**: Seriously evaluate Proposal 2 (queue-worker). Unless organization has deep event-driven experience, simpler pattern may be better choice.

---

## Conclusion and Risk Assessment

### Overall Assessment

**Can this architecture scale to requirements?** YES, but at significantly higher cost than projected.

**Should this architecture be implemented?** CONDITIONAL:
- ✅ **Implement IF**: Organization has event-driven expertise, values loose coupling above all else, and accepts 40% cost premium
- ❌ **Reject IF**: Team lacks event-driven experience, budget is fixed at $93k/month, or operational simplicity is priority

### Final Risk Rating: MEDIUM-HIGH ⚠️

**Justification**:
- Architecture is fundamentally sound but severely underpriced
- Multiple critical design flaws require fixes before implementation
- Operational complexity will be higher than acknowledged
- Cost overrun risk is near-certain without budget revision

### Go/No-Go Recommendation

**Conditional GO** with following prerequisites:
1. ✅ Secure budget increase to $130-140k/month
2. ✅ Eliminate synchronous API pattern
3. ✅ Fix Cosmos DB data residency violation (separate instances per region)
4. ✅ Hire or train SRE with event-driven expertise
5. ✅ Complete 2-week proof of concept validating cost and scalability assumptions

**If any prerequisite cannot be met**: **NO-GO** → Select Proposal 2 instead.

---

**Document Version**: 1.0  
**Analysis Date**: November 21, 2025  
**Analyst**: Senior Principal Engineer, Distributed Systems  
**Focus Areas**: Scalability & Cost Efficiency
