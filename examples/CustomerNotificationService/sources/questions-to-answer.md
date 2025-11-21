# Critical Questions to Answer

## Architecture & Design

1. **What is the overall system architecture?**
   - How do services interact with the notification service?
   - What are the major components and their responsibilities?
   - How do components communicate (sync vs async)?

2. **How do we handle synchronous vs asynchronous notification requests?**
   - Should we support both patterns in the API?
   - How do we handle client timeouts for sync requests?
   - What's the trade-off between simplicity and flexibility?

3. **What queuing strategy should we use?**
   - Single queue vs per-channel queues vs per-priority queues?
   - How do we ensure fair processing across priorities?
   - How do we prevent head-of-line blocking?

4. **How do we handle template storage and versioning?**
   - Where are templates stored (database, blob storage, CDN)?
   - How do we version templates and ensure backward compatibility?
   - How do we validate templates before production use?

## Scalability & Performance

5. **How do we scale to handle peak traffic (2,000 msg/sec)?**
   - What are the auto-scaling triggers and policies?
   - How quickly can we scale up during traffic bursts?
   - What are the scaling limits of each component?

6. **How do we prevent the system from overwhelming downstream providers?**
   - What rate-limiting strategy do we use?
   - How do we handle provider rate limit errors?
   - Should we implement backpressure to slow down upstream services?

7. **What's our strategy for handling 3x growth (150M/day)?**
   - Which components become bottlenecks first?
   - What changes are needed to support 3x volume?
   - Can we scale horizontally or do we need architectural changes?

8. **How do we optimize database performance?**
   - What indexing strategy for preference lookups?
   - How do we partition data for global distribution?
   - Hot partition mitigation for high-volume customers?

## Reliability & Resilience

9. **How do we ensure zero message loss?**
   - What guarantees does the message queue provide?
   - How do we handle partial failures during processing?
   - What happens if a worker crashes mid-processing?

10. **What's the retry strategy for failed notifications?**
    - Exponential backoff parameters (initial delay, max delay, max attempts)?
    - Do we retry different channels (email fails → try SMS)?
    - How do we avoid retry storms?

11. **How do we handle provider outages?**
    - Automatic failover to backup providers?
    - Queue notifications for later retry?
    - What's the customer impact during outage?

12. **What happens during database failures?**
    - Can we serve read traffic from cache?
    - How do we handle preference updates during outage?
    - Degraded mode operations?

## Data Management

13. **Where do we store customer preferences and how do we keep them consistent?**
    - Single source of truth vs distributed caches?
    - How do we handle conflicting updates?
    - Consistency model (strong vs eventual)?

14. **How do we handle data residency requirements (GDPR)?**
    - Regional database deployments?
    - Request routing based on customer location?
    - Cross-region replication policies?

15. **What's our audit logging strategy?**
    - What events do we log?
    - How long do we retain logs?
    - How do we handle log volume (50M notifications/day)?
    - Hot storage vs cold storage?

16. **How do we support data deletion requests (GDPR right to erasure)?**
    - What data do we need to delete?
    - How do we ensure complete deletion across all systems?
    - Timeline to complete deletion?

## Integration & Migration

17. **What does the API contract look like?**
    - REST endpoints and payload structure?
    - Authentication and authorization model?
    - Error response format and codes?
    - Versioning strategy?

18. **How do consuming services migrate to the new notification service?**
    - Can they migrate gradually (service by service)?
    - Do we need to support parallel running (old + new)?
    - What's the rollback plan if migration fails?

19. **How do we handle template migration?**
    - Do we port existing templates or create new ones?
    - Who owns template migration (CNS team or service teams)?
    - Validation and testing process?

20. **What's the impact on consuming services?**
    - Do they need code changes or just configuration?
    - What's the effort to migrate each service?
    - Do we provide migration tools or libraries?

## Operations & Observability

21. **What metrics and monitoring do we need?**
    - Key SLIs (Service Level Indicators)?
    - Alerting thresholds and escalation policies?
    - Dashboard requirements?

22. **How do we debug failed notifications?**
    - Correlation IDs and distributed tracing?
    - Log aggregation and search?
    - Tools for support team to investigate issues?

23. **What's the deployment and rollout strategy?**
    - Blue-green deployment for zero downtime?
    - Canary releases with traffic splitting?
    - Automated rollback triggers?

24. **How do we handle operational tasks?**
    - Template updates without deployment?
    - Customer preference bulk updates?
    - Dead-letter queue processing?

## Cost & Efficiency

25. **How do we optimize costs while maintaining reliability?**
    - Intelligent batching to reduce provider costs?
    - Channel selection based on cost (email before SMS)?
    - Infrastructure right-sizing?

26. **What's the cost per notification at different scales?**
    - Current: 50M/day
    - Future: 150M/day
    - How does cost scale with volume?

27. **Can we reduce SMS costs through smart routing?**
    - Try email first, SMS only if email fails?
    - SMS only for critical notifications?
    - Batch SMS for non-urgent notifications?

## Security & Compliance

28. **How do we secure PII data (emails, phone numbers)?**
    - Encryption at rest and in transit?
    - Access controls and least privilege?
    - Audit logging for PII access?

29. **How do we ensure we respect opt-outs in real-time?**
    - Preference cache strategy?
    - Cache invalidation latency?
    - Fallback if cache is unavailable?

30. **How do we handle authentication and authorization?**
    - Service-to-service authentication (OAuth 2.0)?
    - Rate limiting per consuming service?
    - API key management and rotation?
