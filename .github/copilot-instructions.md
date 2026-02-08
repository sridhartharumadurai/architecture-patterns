# GitHub Copilot Instructions

## Enterprise Integration & Messaging Architecture Standards

This repository follows strict **enterprise architecture patterns** for integration, messaging, and data synchronization.

GitHub Copilot MUST generate code that adheres to the standards defined below.

If generated code violates these standards, prefer the patterns described here.

---

# 1. General Architecture Rules

## Always Prefer

* Event-driven design
* Asynchronous messaging
* Loose coupling between services
* Idempotent processing
* Retry-safe logic
* Horizontal scalability
* Observability (logs + metrics)

## Avoid

* Tight service-to-service coupling
* Blocking network calls inside listeners
* Direct DB-to-DB integrations
* Shared databases across services
* Synchronous chains across multiple services
* Manual thread management unless required
* Auto-commit Kafka consumers

---

# 2. Integration Pattern Guidelines

## Synchronous APIs (REST/gRPC)

Use ONLY when:

* Immediate response required
* UI request/response
* Simple lookup/CRUD

Copilot should:

* Use Spring REST controllers
* Add timeouts
* Use circuit breakers (Resilience4j)
* Keep calls short and lightweight

Never:

* Chain more than 2 downstream calls
* Use synchronous APIs for heavy work

---

## Asynchronous Messaging (Default Choice)

Use for:

* Background work
* Cross-service communication
* Notifications
* Processing pipelines

Copilot should:

* Use Kafka or message queues (SQS/SNS on AWS)
* Publish domain events
* Keep producers thin
* Use consumers for processing

---

## Cloud-Native Integration Stack (AWS + Kong + PingID)

> Full specification: `docs/integration-pattern-standard.md`

### Canonical Inbound Flow

```
Client → Kong API Gateway → PingID (OIDC token validation) → AWS Backend (Private Subnet)
```

### Kong API Gateway Rules

* Kong is the **single internet-facing ingress** — no backend is directly exposed
* Use the **OpenID Connect plugin** for JWT validation against PingID JWKS
* Apply plugins in this order: OIDC → rate-limiting → request-transformer → correlation-id → logging
* Upstream headers must include: `X-Correlation-ID`, `X-Consumer-ID`, `X-Authenticated-Scope`
* Kong admin API must **never** be internet-accessible

### PingID / OIDC AuthN/AuthZ Rules

* User flows: **Authorization Code + PKCE**
* Service-to-service: **Client Credentials** grant
* Token format: Signed JWT (RS256), 15-min access token lifetime
* Required claims: `sub`, `aud`, `iss`, `exp`, `scope`, `roles`
* Scope-to-route mapping convention:

| Route Pattern       | Scope            | Methods          |
| ------------------- | ---------------- | ---------------- |
| `/api/v1/<domain>`  | `<domain>.read`  | GET              |
| `/api/v1/<domain>`  | `<domain>.write` | POST, PUT, PATCH |
| `/api/v1/<domain>`  | `<domain>.admin` | DELETE           |

### AWS Backend Rules

* **IAM roles only** — never embed AWS access keys in Kong or Lambda config
* Kong connects to AWS via **VPC Link** (ALB/NLB) or the **aws-lambda plugin** with execution role
* Lambda functions run in **private subnets** with NAT Gateway for outbound
* SQS: visibility timeout = 6× Lambda timeout; DLQ after 3 retries; batch size 10
* SNS for fan-out; SQS subscriptions per downstream consumer
* RDS via **RDS Proxy** with IAM DB authentication; never direct connections from Lambda
* All resources tagged: `team`, `environment`, `cost-center`, `data-classification`

### AWS Resource Naming

```
Lambda:  <team>-<domain>-<action>-fn       → payments-order-process-fn
SQS:     <team>-<domain>-<action>-queue    → payments-order-process-queue
DLQ:     <team>-<domain>-<action>-dlq      → payments-order-process-dlq
SNS:     <team>-<domain>-<event>-topic     → payments-order-created-topic
RDS:     <team>-<domain>-db                → payments-orders-db
```

### Kong Service/Route Naming

```
Service:  svc-<domain>-<version>           → svc-orders-v1
Route:    rt-<domain>-<action>-<version>   → rt-orders-list-v1
```

---

# 3. Kafka Standards (MANDATORY)

All generated Kafka code MUST follow these rules.

## Producer Requirements

* Idempotent producer enabled
* acks=all
* Compression enabled (lz4 or zstd)
* Use keys for partitioning
* Schema-based messages (Avro/JSON schema)

## Consumer Requirements

* enable-auto-commit = false
* Manual acknowledgment
* Idempotent logic
* Retry with backoff
* Dead Letter Topic configured
* No blocking operations inside listener thread

---

# 4. Spring Boot Kafka Implementation Rules

Copilot MUST follow these patterns when generating consumers.

---

## Default Consumer Pattern (Preferred)

Use:
Partition-based concurrency with Spring Kafka

### Configuration

```yaml
spring.kafka.listener.concurrency = <number_of_partitions>
spring.kafka.consumer.enable-auto-commit = false
```

### Example

```java
@KafkaListener(topics = "orders", containerFactory = "kafkaListenerContainerFactory")
public void consume(OrderEvent event, Acknowledgment ack) {
    service.process(event);
    ack.acknowledge();
}
```

---

## High Throughput Pattern

If processing involves:

* database writes
* external calls
* heavy IO

Copilot should generate BATCH consumers:

```java
@KafkaListener(topics = "orders")
public void consume(List<OrderEvent> events) {
    service.bulkProcess(events);
}
```

Never:

* Process 1 record at a time for heavy DB operations

---

## CPU-Heavy Pattern

If compute-heavy logic exists:

Copilot should:

* delegate to ExecutorService or async pool
* keep listener non-blocking

Example:

```java
executor.submit(() -> process(event));
```

---

## Exactly-Once Pattern (only when explicitly required)

Use:

* Transactions
* Kafka + DB atomic commits

Do NOT use transactions by default due to performance cost.

---

# 5. Threading Rules

Copilot must:

## Allowed

* Spring Kafka concurrency
* ExecutorService pools
* @Async for background tasks

## Not Allowed

* Manual thread creation in listener
* Blocking sleep calls
* Long-running tasks inside poll thread

---

# 6. Data Synchronization Rules

Copilot should choose:

| Scenario          | Pattern               |
| ----------------- | --------------------- |
| Real-time request | REST                  |
| Service events    | Kafka pub/sub         |
| Large volumes     | Batch/ETL             |
| Replication       | CDC                   |
| Analytics         | Stream/batch pipeline |

Never:

* Query another service’s database directly
* Use synchronous polling loops

---

# 7. Reliability Patterns (Always Include)

Copilot must generate:

* Retries with exponential backoff
* Dead Letter Topic (Kafka) / Dead Letter Queue (SQS)
* Structured logging
* Metrics (Micrometer for Spring; CloudWatch Embedded Metrics for Lambda)
* Correlation IDs (`X-Correlation-ID` propagated end-to-end)
* Idempotency keys
* Circuit breakers (Resilience4j for Spring; retry config for Lambda event sources)
* RFC 7807 Problem Details for all error responses

---

# 8. Observability Standards

Generated code must include:

* Slf4j logging (Spring) / structured JSON logging (Lambda)
* Error logs with context
* Metrics counters/timers
* Consumer lag monitoring hooks (Kafka) / `ApproximateAgeOfOldestMessage` alarms (SQS)
* X-Ray tracing enabled for Lambda and API Gateway

Example (Spring):

```java
log.info("Processing orderId={} correlationId={}", orderId, correlationId);
```

Example (Lambda / Python):

```python
logger.info("Processing", extra={"orderId": order_id, "correlationId": correlation_id})
```

---

# 9. Naming Conventions

Topics:

```
domain.event.v1
order.created.v1
payment.completed.v1
```

Consumer groups:

```
<service-name>-consumer
```

Classes:

```
OrderEventConsumer
OrderEventProducer
OrderProcessingService
```

---

# 10. Code Generation Preferences

When generating code:

Prefer:

* Constructor injection
* Service layer separation
* Stateless consumers
* Small methods
* Clear exception handling

Avoid:

* Business logic inside controllers/listeners
* God classes
* Static state
* Tight coupling between modules

---

# 11. Default Decision Rule for Copilot

If unsure:

* Integration → Kafka (on-prem/hybrid) or SNS+SQS (AWS-native)
* Processing → Async
* High load → Batch
* Scaling → Partitions (Kafka) or Lambda concurrency (AWS)
* Replication → CDC
* Critical correctness → Transactions
* API ingress → Kong API Gateway
* AuthN/AuthZ → PingID OIDC (never custom token logic)
* AWS connectivity from gateway → VPC Link + IAM roles (never access keys)

---

# 12. Key Reference Documents

| Document | Path | Covers |
| -------- | ---- | ------ |
| Integration Pattern Standard | `docs/integration-pattern-standard.md` | Kong + PingID + AWS architecture, Draw.io diagram spec, security checklist |

---

# 13. Summary

Copilot should generate:

* Event-driven
* Asynchronous
* Scalable
* Fault-tolerant
* Idempotent
* Observable
* Cloud-native code
* Kong-routed with PingID OIDC enforcement (for API ingress)
* IAM-role-based AWS integration (never hardcoded credentials)

Never generate tightly coupled, blocking, synchronous-heavy, or credential-embedded designs.

---

End of Instructions
