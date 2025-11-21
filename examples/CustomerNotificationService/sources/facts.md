# Facts and Calculations

## Scale and Volume Data

### Current State (15 Services Combined)
- **Daily notification volume**: 38M per day
- **Peak hour**: 3.2M per hour (08:00-09:00 UTC, morning check-ins)
- **Peak second**: ~1,200 per second during flash sales
- **Channel breakdown**:
  - Email: 25M/day (66%)
  - SMS: 8M/day (21%)
  - Push: 5M/day (13%)
  - In-app: N/A (new capability)

### Target State (Year 1)
- **Daily volume**: 50M per day (31% growth)
- **Average throughput**: 580 notifications/second
- **Peak throughput**: 2,000 notifications/second (3.4x average)
- **Growth runway**: Design for 150M per day (3x target)

### Customer Base
- **Active customers**: 12M globally
- **Email addresses**: 11.5M (96% have email)
- **Phone numbers**: 8M (67% have SMS-capable number)
- **Push registrations**: 6M (50% have mobile app)
- **Geographic distribution**:
  - North America: 5M (42%)
  - Europe: 4M (33%)
  - Asia Pacific: 2M (17%)
  - Other: 1M (8%)

## Performance Benchmarks

### Current System Performance
- **Email delivery time**: P50 15s, P95 45s, P99 120s
- **SMS delivery time**: P50 3s, P95 8s, P99 20s
- **Push delivery time**: P50 2s, P95 5s, P99 15s
- **API response time** (async): P50 80ms, P95 250ms, P99 800ms
- **Failed delivery rate**: 8% (mostly invalid contact info)

### Provider SLAs and Limits

#### SendGrid
- **Rate limit**: 10,000 emails/second per account
- **Reliability**: 99.9% uptime SLA
- **Average latency**: Accept to send ~5 seconds
- **Pricing**: $0.0001 per email (volume tier)
- **Bounce rate**: Industry average 2-3%

#### Twilio
- **Rate limit**: 1,000 SMS/second per account
- **Reliability**: 99.95% uptime SLA
- **Average latency**: Submit to delivery ~3 seconds
- **Pricing**: $0.0075 per SMS (US), $0.02-0.10 international
- **Delivery rate**: 95-98% (carrier dependent)

#### Azure Notification Hubs
- **Rate limit**: 10,000 sends/second per hub
- **Reliability**: 99.9% uptime SLA
- **Average latency**: <2 seconds to platform (iOS APNs, Android FCM)
- **Pricing**: $0.00001 per notification
- **Delivery rate**: 90-95% (device availability dependent)

## Cost Estimates

### Current State (Distributed Across Services)
- **SendGrid**: $2,500/month (25M emails × $0.0001)
- **Twilio**: $60,000/month (8M SMS × $0.0075)
- **Azure Notification Hubs**: $50/month (5M push × $0.00001)
- **Infrastructure**: ~$8,000/month (15 services, inefficient sizing)
- **Total**: $70,550/month

### Target State Projection (Centralized)

#### Notification Provider Costs
- **SendGrid**: $3,300/month (33M emails × $0.0001)
- **Twilio**: $78,750/month (10.5M SMS × $0.0075)
- **Azure Notification Hubs**: $65/month (6.5M push × $0.00001)
- **Subtotal**: $82,115/month

#### Azure Infrastructure Costs (Estimated)

**API Layer (AKS)**
- 5 pods (3 normal + 2 peak scale), Standard_D4s_v3 nodes
- Cost: $600/month

**Message Queue (Azure Service Bus Premium)**
- 2 messaging units (8 MB/sec throughput)
- Cost: $1,340/month

**Database (Cosmos DB)**
- 10,000 RU/s provisioned, multi-region (US, EU)
- 500 GB storage
- Cost: $5,840/month

**Storage (Azure Blob)**
- Template storage, audit logs
- 1 TB hot tier
- Cost: $50/month

**API Management**
- Premium tier (required for VNET integration, multi-region)
- Cost: $2,800/month

**Monitoring & Logging**
- Application Insights, Log Analytics
- Cost: $400/month

**Infrastructure Subtotal**: $11,030/month

### Total Estimated Cost
- **Providers**: $82,115/month
- **Infrastructure**: $11,030/month
- **Total**: $93,145/month

**Cost Impact**: +$22,595/month (+32%) vs current state

**Note**: While total cost increases, cost per notification decreases due to efficiency gains, and this centralizes costs for better visibility and optimization opportunities.

### Cost Optimization Opportunities
- Intelligent batching could reduce email costs by 15-20%
- Smart channel selection (email before SMS) could reduce SMS costs by 20-30%
- Potential savings: $18,000-25,000/month
- **Optimized total**: $68,000-75,000/month (approaching cost-neutral)

## Performance Calculations

### Queue Sizing

**Average throughput**: 580 msg/sec
**Peak throughput**: 2,000 msg/sec
**Average processing time**: 5 seconds per notification

**Queue depth during peak**:
- Incoming: 2,000 msg/sec
- Processing capacity needed: 2,000 msg/sec
- Burst capacity: 2,000 × 30 seconds = 60,000 messages
- Recommended queue capacity: 100,000 messages (safety margin)

### Database Sizing

**Customer preferences**: 12M records × 2 KB = 24 GB
**Template library**: ~500 templates × 50 KB = 25 MB
**Delivery logs** (90 days retention):
- 50M/day × 90 days × 1 KB = 4.5 TB
- With compression: ~1 TB

**Total storage**: ~1.1 TB
**RU requirements**:
- Preference lookups: 580/sec × 10 RU = 5,800 RU/sec
- Preference updates: 10/sec × 20 RU = 200 RU/sec
- Log writes: 580/sec × 5 RU = 2,900 RU/sec
- Total: ~9,000 RU/sec + overhead = 10,000 RU/sec provisioned

### Network Bandwidth

**API ingress**: 580 req/sec × 2 KB = 1.16 MB/sec (9.3 Mbps)
**Provider egress**: 580 req/sec × 3 KB = 1.74 MB/sec (13.9 Mbps)
**Peak**: 3x = ~42 Mbps

**Monthly bandwidth**: 4.5 TB (well within Azure limits)

## Known System Characteristics

### Notification Patterns
- **Time-of-day variation**: 5x higher volume 8am-10am vs 2am-4am
- **Day-of-week variation**: Weekdays 1.5x higher than weekends
- **Seasonal spikes**: Black Friday 10x normal volume
- **Critical notification ratio**: 5% of total volume (must be fast)

### Failure Modes (Current State)
- **Invalid email**: 3% of email attempts
- **Invalid phone number**: 5% of SMS attempts
- **Unregistered devices**: 10% of push attempts
- **Provider outages**: ~0.1% of time (per provider)
- **Transient failures**: 2% retry successfully

### Regulatory Requirements
- **Opt-out latency**: Customer expects immediate effect (< 1 minute)
- **Data residency**: EU customers must have data in EU (GDPR)
- **Audit retention**: 90 days minimum, 7 years for financial notifications
- **Unsubscribe link**: Required in all marketing emails (CAN-SPAM)
