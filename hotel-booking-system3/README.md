# Hotel Booking System

# Hotel Management System — Microservices Architecture

**Stack:** .NET Core (C#) microservices, React front-end, Kafka (Confluent/Apache) event bus, SQL Server (RDBMS), Redis (cache), Kubernetes, Prometheus/Grafana/Jaeger, Elasticsearch/Fluentd/Kibana (EFK), IdentityServer/Duende (OIDC/OAuth2), Ocelot/YARP (API Gateway), Istio/Linkerd (Service Mesh), WAF + DDoS protection, Vault/Secrets Manager, xUnit for tests.

---

## Goals

* Provide **hotel search**, **get room price**, **process payment**, and on success **finalize booking**.
* Microservice architecture with **loose coupling**, **observability**, **security** (authentication/authorization), **DDoS detection**, **WAF/web shield**, **distributed logging**, **caching**, and **streaming/event-driven flows**.

---

## High-level components

1. **React SPA (Web UI)**

   * Calls API Gateway. Handles client authentication (OIDC code flow + PKCE), shows search, booking flow, payment UI (redirect to payment provider or host payment UI).

2. **API Gateway (Ocelot / YARP on K8s)**

   * Single entry point for clients.
   * Responsibilities: routing, authentication validation (JWT introspection / OIDC token validation), rate limiting, IP allowlist/blacklist, basic WAF integration hooks, circuit-breaker at edge, request/response logging.

3. **Auth & Identity (Identity Service)**

   * IdentityServer / Duende / Azure AD B2C depending on infra.
   * Issues JWTs (access + refresh) and supports OAuth2/OIDC.
   * Stores user credentials and roles (RBAC). Integrates with 2FA, social logins if required.

4. **Hotel Search Service**

   * Read-optimized service; uses SQL Server (search index tables) and Redis cache.
   * Exposes endpoint: `/search?city=&checkin=&checkout=&guests=`.
   * Optionally prebuild search indexes or integrate with ElasticSearch for advanced search.

5. **Pricing Service**

   * Responsible for calculating room price (dynamic rules, occupancies, promotions).
   * Takes input: hotelId, roomType, date range, promo code, currency.
   * Publishes pricing events (e.g., `pricing.calculated`) and caches computed quotes in Redis (quote TTL).

6. **Booking Service**

   * Manages booking lifecycle (draft -> reserved -> confirmed -> cancelled).
   * Uses SQL Server for authoritative booking store.
   * Implements **Saga pattern** for the booking transaction using **choreography** via Kafka topics or **orchestration** via a Saga Orchestrator service.
   * Ensures idempotency keys for repeated submissions.

7. **Payment Service**

   * Integrates with external payment gateway (Stripe, Adyen, Razorpay, etc.).
   * Handles payment initiation, webhook handling, reconciliation.
   * Publishes events: `payment.authorized`, `payment.failed`, `payment.captured`.
   * On success, emits `booking.confirm` (or notifies Booking Service via a Kafka event to complete the saga).

8. **Inventory/Rooms Service**

   * Manages room availability and lock/unlock operations for provisional holds.
   * Exposes idempotent APIs for hold/release/confirm.

9. **Notification Service**

   * Sends emails/SMS on booking confirmation, payment failure, etc. Subscribes to booking/payment topics.

10. **Admin Service**\n   - For hotel managers: manage hotels, room types, availability, pricing rules.

11. **Event Bus (Kafka)**

* Topics: `hotel.search.logs` (optional), `pricing.calculated`, `booking.requested`, `booking.hold`, `booking.confirmed`, `payment.requested`, `payment.completed`, `inventory.updated`, `notifications.*`.
* Use **outbox pattern** in each service to reliably publish events when updating their DB.

12. **Cache (Redis)**

* Caching search results, pricing quotes, sessions, rate-limiting counters.

13. **Distributed Log / Observability**

* EFK stack for logs (Elasticsearch + Fluentd + Kibana).
* Prometheus for metrics, Grafana for dashboards, Jaeger for distributed tracing.
* Correlate traces with request IDs or `baggage` across services; inject trace id at API Gateway.

14. **Service Mesh (Istio / Linkerd)**

* mTLS between services, fine-grained traffic policies, automatic telemetry ingestion, traffic shifting, fault-injection for testing.
* Use mesh telemetry exported to Prometheus/Jaeger.

15. **WAF & DDoS Protection**

* WAF (ModSecurity or cloud-managed WAF e.g., Azure Front Door WAF, Cloudflare WAF) in front of API Gateway.
* DDoS detection & mitigation (Cloudflare, AWS Shield, Azure DDoS) with rate-limiting thresholds and automatic IP blocking/geo-blocking.

16. **Secrets / Config**

* Use Vault (HashiCorp) or cloud secrets manager; K8s external-secrets to inject at runtime.

17. **CI/CD**

* Build pipelines per service (GitHub Actions / Azure DevOps / GitLab CI). Image scanning, unit/integration tests (xUnit), push to container registry, Helm charts, automated canary deploys via service mesh.

---

## Data Modeling (brief)

* Each microservice owns its database (SQL Server schema per service). Do NOT share DBs across services.
* Use Redis for ephemeral caches and locks.
* Use Outbox table in each service DB for dependable event publishing (transactional outbox).

### Example DBs

* `HotelSearchDB` (Hotels, Rooms, Images, Amenities)
* `PricingDB` (Rules, Promotions)
* `BookingDB` (Bookings, BookingLineItems, PaymentRef, Status, IdempotencyKey, Outbox)
* `InventoryDB` (RoomInventory, Holds)
* `UserDB` (Users, Roles)

---

## Booking sequence (typical flow)

1. User searches on React -> API Gateway -> Hotel Search Service (cache hit or DB) -> results.
2. User picks room -> front-end calls Pricing Service (`/price`) -> Pricing Service calculates and stores a temporary quote (`quoteId`) in Redis with TTL.
3. User initiates booking -> Booking Service creates a `Draft` booking (DB) + writes outbox event `booking.requested`.
4. Booking Service publishes event; Inventory Service receives to place a hold on rooms.
5. Front-end calls Payment Service to pay (or Payment Service handles redirect); Payment Service creates payment intent and publishes `payment.requested`.
6. On payment success webhook, Payment Service publishes `payment.completed`.
7. Booking Service, listening to `payment.completed`, verifies payment and transitions booking to `Confirmed`, releases the hold, updates DB, and publishes `booking.confirmed`.
8. Notification Service sends confirmation.

Use **compensation actions** for rollback (release inventory, refund) if later steps fail.

---

## Reliability & Resilience

* **Retries** with exponential backoff and jitter; use Polly in .NET clients.
* **Circuit Breaker** per downstream dependency.
* **Bulkheads**: limit concurrency per downstream call.
* **Idempotency**: Payment endpoints and booking endpoints use idempotency-key header.
* **Outbox + Debezium** for CDC if using event-driven DB replication.
* **Dead-letter topics** in Kafka for failed messages.

---

## Security

* **Auth**: OAuth2/OIDC with IdentityServer/managed provider. Access tokens are JWTs.
* **AuthZ**: Role-based checks and claim-based permission checks inside services.
* **mTLS** in service mesh + network policies to limit pod-to-pod access.
* **API Gateway** validates tokens and enforces rate limits and quotas.
* **WAF** for common OWASP top 10 protections.
* **Secrets encryption** at rest and transit.
* **PCI**: Do NOT store full card details. Use tokenization via payment provider.

---

## Observability & Security Detection

* **Tracing**: instrument services with OpenTelemetry; forward to Jaeger.
* **Metrics**: Prometheus metrics for request latencies, error rates, queue lag, Kafka consumer lag, Redis hit rate.
* **Logs**: structured logs (JSON) with correlation ids and spans; ship via Fluentd to Elasticsearch.
* **Alerts**: alert on SLO breaches, high error rates, slow DB queries, Kafka lags, sudden traffic spikes.
* **DDoS detection**: integrate WAF and cloud DDoS; set anomaly detection rules and automated mitigations.
* **Bot and Abuse detection**: rate-limiting, dynamic challenges (CAPTCHA), and IP reputation lists.

---

## Service Mesh + Observability Integration

* Istio provides fine-grained telemetry, mTLS, and resilience.
* Use Istio's metrics alongside Prometheus; use Jaeger for distributed traces.
* Mesh can enforce policies (e.g., only allow Booking Service to call Inventory Service).

---

## Kafka topics (examples)

* `pricing.quote.created` (key: quoteId)
* `booking.requested` (key: bookingId)
* `inventory.hold` (key: roomId)
* `payment.completed` (key: paymentId)
* `booking.confirmed` (key: bookingId)

---

## Testing strategy

* Unit tests with **xUnit**, mocking Kafka producers/consumers and DB.
* Integration tests spinning up test SQL Server (or use TestContainers), Redis, and a lightweight Kafka (or embedded mocks).
* Contract tests for API boundaries (Pact).
* End-to-end tests for critical flows (search -> price -> pay -> confirm).

---

## Deployment & Infra

* Kubernetes cluster (AKS/EKS/GKE or self-hosted). Use Helm charts for services.
* Use Horizontal Pod Autoscaler (HPA) based on CPU and custom Prometheus metrics (request rate, error rate).
* Use PodDisruptionBudgets for stateful services.
* Backups: regular SQL Server backups and retention policies.

---

## Non-functional requirements & SLOs (examples)

* Search 95th percentile latency < 300ms.
* Pricing calculation < 500ms.
* Payment end-to-end less than 3s (depends on gateway).
* 99.9% availability for critical services.

---

## Implementation Roadmap (suggested phases)

1. Core services: Hotel Search, Pricing, Booking, Inventory, Payment (stubs), API Gateway, Identity Service.
2. Integrate Kafka (local dev), Redis caching, SQL Server schemas, outbox.
3. Add Observability (Prometheus + Jaeger) and distributed logging.
4. Deploy to Kubernetes; add service mesh.
5. Add WAF, DDoS protection, and hardened security.
6. Add CI/CD, automated tests, and run chaos experiments.

---

## Artifacts I can produce next (pick one)

* Detailed sequence diagrams (PlantUML).
* Kubernetes manifests / Helm charts for one or more services.
* Sample .NET Core microservice template (controller, repository, outbox, Kafka publisher, xUnit tests).
* API contract (OpenAPI) for each service.

