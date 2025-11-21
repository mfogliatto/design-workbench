# Glossary

## Key Terms

### System Components

**Customer Notification Service (CNS)**
The centralized microservice responsible for sending notifications across multiple channels. Primary subject of this architecture design.

**Consuming Service**
Any internal microservice that needs to send notifications to customers (e.g., Order Service, Account Service, Promotion Service). CNS clients.

**Notification Provider**
External third-party service that delivers notifications to end users (SendGrid for email, Twilio for SMS, Azure Notification Hubs for push).

**Template Engine**
Component responsible for rendering notification content by merging templates with customer data/variables.

**Preference Service**
Component that stores and manages customer notification preferences (channels, opt-outs, quiet hours).

### Notification Concepts

**Notification Channel**
The medium through which a notification is delivered: email, SMS, push notification, or in-app message.

**Notification Priority**
Classification of notification urgency: critical (security alerts), high (order confirmations), normal (promotions), low (newsletters).

**Notification Template**
Pre-defined message structure with placeholders for dynamic content. Example: "Hello {{firstName}}, your order {{orderId}} has shipped."

**Tracking ID**
Unique identifier assigned to each notification request for status tracking and audit purposes.

**Delivery Status**
Current state of a notification: queued, sent, delivered, failed, bounced, or rejected.

### Customer Preferences

**Opt-Out**
Customer's explicit request to stop receiving certain types of notifications. Can be global (all notifications) or category-specific (marketing only).

**Quiet Hours**
Time window during which non-critical notifications should not be sent to respect customer preferences (typically 10pm-8am local time).

**Channel Preference**
Customer's preferred notification channel. Example: prefer email over SMS for non-urgent messages.

**Contact Method**
Specific destination for notifications: email address, phone number, device push token, or user ID for in-app messages.

### Delivery Patterns

**Synchronous Notification**
API pattern where the caller waits for confirmation that the notification was accepted/sent before proceeding. Higher latency but provides immediate feedback.

**Asynchronous Notification**
API pattern where the caller receives immediate acknowledgment (tracking ID) and continues processing. Notification is queued for background delivery.

**Batch Notification**
Single API request that sends the same notification to multiple recipients (up to 1000). More efficient than individual requests.

**Scheduled Notification**
Notification that should be delivered at a specific future time rather than immediately.

**Recurring Notification**
Notification that repeats on a schedule (daily digest, weekly summary). Managed by the consuming service triggering CNS repeatedly.

### Reliability Concepts

**At-Least-Once Delivery**
Guarantee that every notification will be delivered one or more times. May result in duplicates during retry scenarios.

**Exactly-Once Delivery**
Guarantee that every notification will be delivered exactly once with no duplicates. Much harder to achieve than at-least-once.

**Idempotency**
Property where processing the same notification request multiple times has the same effect as processing it once. Prevents duplicate sends.

**Retry Logic**
Automated mechanism to reattempt delivery of failed notifications with delays between attempts (typically exponential backoff).

**Dead-Letter Queue (DLQ)**
Storage for notifications that have failed repeatedly and exceeded maximum retry attempts. Requires manual investigation.

**Circuit Breaker**
Pattern that temporarily stops requests to a failing provider to prevent cascading failures and allow recovery.

### Performance Concepts

**Throughput**
Number of notifications processed per unit of time. Measured in messages per second (msg/sec) or millions per day.

**Latency**
Time delay from notification request to delivery. Measured at P50 (median), P95 (95th percentile), P99 (99th percentile).

**Queue Depth**
Number of notifications waiting to be processed. High queue depth indicates backlog or insufficient processing capacity.

**Backpressure**
Mechanism to slow down incoming requests when the system is overloaded, preventing queue overflow and resource exhaustion.

**Rate Limiting**
Constraint on how many requests a provider will accept per time period. Example: SendGrid allows 10,000 emails/second.

### Data Concepts

**PII (Personally Identifiable Information)**
Data that can identify an individual: email addresses, phone numbers, names. Subject to privacy regulations (GDPR, CCPA).

**Data Residency**
Requirement that certain data must be stored in specific geographic regions. Example: EU customer data must stay in EU (GDPR compliance).

**Right to Erasure**
GDPR requirement allowing customers to request deletion of their personal data within 30 days.

**Audit Trail**
Comprehensive log of all actions taken on customer preferences and notification deliveries for compliance and debugging.

### Provider-Specific Terms

**SendGrid**
Third-party email service provider. Handles email delivery at scale with bounce management and reputation monitoring.

**Twilio**
Third-party SMS service provider. Delivers text messages globally through carrier integrations.

**Azure Notification Hubs**
Microsoft Azure service for sending push notifications to mobile devices (iOS, Android) through APNs and FCM.

**APNs (Apple Push Notification service)**
Apple's infrastructure for delivering push notifications to iOS devices.

**FCM (Firebase Cloud Messaging)**
Google's infrastructure for delivering push notifications to Android devices.

**Bounce**
Email that cannot be delivered due to invalid address or recipient's mailbox being full. Hard bounce (permanent) vs soft bounce (temporary).

**Carrier**
Telecommunications company that delivers SMS messages (AT&T, Verizon, Vodafone, etc.). Different carriers have different reliability.

### Compliance & Regulation

**GDPR (General Data Protection Regulation)**
EU privacy law requiring data protection, consent management, and right to erasure. Applies to EU customers regardless of company location.

**CCPA (California Consumer Privacy Act)**
California privacy law giving consumers rights over their personal data. Applies to California residents.

**CAN-SPAM Act**
US law regulating commercial email. Requires unsubscribe links and accurate sender information in marketing emails.

**Transactional Notification**
Notification required for service operation: order confirmations, password resets, security alerts. Cannot be opted out of (except by closing account).

**Marketing Notification**
Promotional or non-essential notification: sales, newsletters, product recommendations. Must provide easy opt-out mechanism.

### Azure Services

**AKS (Azure Kubernetes Service)**
Managed Kubernetes platform for running containerized applications. CNS will run on AKS.

**Azure Service Bus**
Enterprise message broker for reliable asynchronous communication. Will be used for internal message queuing.

**Azure Cosmos DB**
Globally distributed NoSQL database with multi-region replication. Will store customer preferences and audit logs.

**Azure API Management (APIM)**
API gateway for managing, securing, and monitoring APIs. Will front CNS API.

**Application Insights**
Azure monitoring service for tracking application performance, logs, and distributed traces.
