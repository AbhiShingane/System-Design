# Microservices Architecture — Complete System Design Interview Guide

> Covers every concept from the book, with pseudocode/C++ examples, interview Q&A, and architecture diagrams explained. Structured from core theory → architecture → patterns → security → testing.

---

## 1. Core Design Principles

### 1.1 Cohesion

**Definition:** The degree to which elements of a module *belong together* and serve a single, focused purpose.

**Key rule:** High Cohesion → each microservice does **one thing and does it well**.

**Low vs High Cohesion Example:**

```
LOW COHESION (Monolith):
+-------------------------------------------+
|  eShop (single process)                   |
|  - Order Management                       |
|  - Inventory Management                   |
|  - User Management                        |
|  - Payment Processing                     |
|  - Marketing & Demand Generation          |
+-------------------------------------------+

HIGH COHESION (Microservices):
+--------------------+  +--------------------+
| Order Service      |  | Inventory Service  |
| - Create order     |  | - Update stock     |
| - Cancel order     |  | - Check quantity   |
+--------------------+  +--------------------+
+--------------------+  +--------------------+
| Payment Service    |  | User Service       |
| - Process payment  |  | - Create account   |
| - Issue refunds    |  | - Login/Logout     |
+--------------------+  +--------------------+
```

**Benefits of High Cohesion:**
- Reduced module complexity
- Easier maintainability (changes affect fewer modules)
- Higher module reusability
- Easier to test in isolation

**Interview Answer:** "High cohesion means each microservice is focused on one business capability. When a change is needed, only one service is affected, enabling independent deployment and team autonomy."

---

### 1.2 Coupling

**Definition:** The degree of *dependency* that exists between two software components.

**Types:**

| Type | Description | Example |
|---|---|---|
| Tight Coupling | High dependency; change in A breaks B | Shared database across services |
| Loose Coupling | Low dependency; services are autonomous | REST API between services |

**Rule:** High Cohesion almost always correlates with Loose Coupling.

```
// TIGHT COUPLING (Anti-pattern):
// Service B directly reads Service A's database table
ServiceB {
    getOrderDetails() {
        query = "SELECT * FROM service_a.orders WHERE id = ?"
        // ❌ B depends on A's internal schema
    }
}

// LOOSE COUPLING (Correct):
// Service B calls Service A's public API
ServiceB {
    getOrderDetails(orderId) {
        response = httpClient.get("http://order-service/api/orders/" + orderId)
        // ✅ Only coupled to the API contract, not internal implementation
    }
}
```

---

### 1.3 Immutability in Microservices

**Definition:** Once a microservice is deployed, the infrastructure it runs on is **never modified in place** — it is replaced entirely with a new version.

**How Docker enables this:**
```
Deploy v1.0 → Immutable Container [v1.0]
Bug fix needed? → Don't patch the running container
Instead:
  Build new image → Immutable Container [v1.1]
  Stop [v1.0], Start [v1.1]
```

**Benefits:**
- Guaranteed identical behavior across dev/stage/production
- Eliminates "it works on my machine" issues
- Simplifies rollback (just re-launch the old image)

---

### 1.4 Open/Close Principle

"Classes should be **open for extension** but **closed for modification**."

```cpp
// Closed for modification (don't change this):
class NotificationService {
public:
    virtual void send(const string& message) = 0;
};

// Open for extension (add new types without modifying existing):
class EmailNotification : public NotificationService {
public:
    void send(const string& message) override {
        cout << "Email: " << message << endl;
    }
};

class SMSNotification : public NotificationService {
public:
    void send(const string& message) override {
        cout << "SMS: " << message << endl;
    }
};
// ✅ Added SMS without changing NotificationService
```

---

### 1.5 DRY — Don't Repeat Yourself

**Concept:** Avoid duplicating code/logic across services.

**In Microservices Context:** DRY applies to **technical utilities**, NOT to domain models.

```
✅ SHARE (Technical utilities):
- Versioned JAR: ServiceResponse<T> wrapper
- Versioned JAR: Custom RestTemplate with TLS handling
- Versioned JAR: Common logging utilities

❌ DO NOT SHARE (Domain models violate Bounded Context):
- A single unified "Customer" class across all services
  (Each service should have its own Customer with only its relevant fields)
```

---

### 1.6 SOLID Principles

| Principle | Key Idea | Microservices Application |
|---|---|---|
| **S** – Single Responsibility | A class has one reason to change | Each microservice owns one business capability |
| **O** – Open/Closed | Open for extension, closed for modification | Add new endpoints without changing existing ones |
| **L** – Liskov Substitution | Subtypes replace supertypes safely | Service contracts are backward-compatible |
| **I** – Interface Segregation | Many specific interfaces > one general interface | Microservices expose focused APIs |
| **D** – Dependency Inversion | Depend on abstractions, not concretions | Services use interface contracts (REST/events), not direct DB access |

---

### 1.7 Single Responsibility Principle (SRP)

"A class should have **only one reason to change**." — Robert C. Martin

**In Microservices:** Apply SRP at the *service boundary* level.

```
BAD — Reporting module doing two things:
+------------------------------------+
| ReportingModule                    |
| - generateReportContent()  ← reason 1: report format changes |
| - sendEmailReport()        ← reason 2: email server changes  |
+------------------------------------+

GOOD — Two separate services:
+----------------------+    +--------------------+
| ReportContentService |    | EmailService       |
| - generate()         | →  | - send(content)    |
+----------------------+    +--------------------+
```

**Interview Tip:** "We partition microservices by applying SRP at the architecture level. Each service changes for one business reason only — enabling independent deployment cycles."

---

## 2. 8 Fallacies of Distributed Computing

Peter Deutsch (1994) + James Gosling (1997) identified 8 **false assumptions** engineers make about distributed systems:

| # | Fallacy | Reality & Mitigation |
|---|---|---|
| 1 | The network is reliable | Networks fail; use retries, circuit breakers |
| 2 | Latency is zero | Network adds latency; use async messaging |
| 3 | Bandwidth is infinite | Minimize payload size; use compression |
| 4 | The network is secure | Encrypt with HTTPS/TLS; use OAuth2+JWT |
| 5 | Topology doesn't change | Use service discovery (Eureka/Consul) |
| 6 | There is one administrator | Use DevOps culture, automated operations |
| 7 | Transport cost is zero | Asynchronous messaging reduces round-trips |
| 8 | The network is homogeneous | Use platform-agnostic protocols (REST, AMQP) |

**Interview Answer:** "These 8 fallacies are why we need circuit breakers (fallacy 1), service discovery (fallacy 5), OAuth2/TLS (fallacy 4), and async messaging (fallacy 2, 7). Spring Cloud directly addresses all 8."

---

## 3. CAP Theorem

**Theorem:** A distributed data store can only simultaneously guarantee **two of three**:

```
              Consistency (C)
             /               \
            /                 \
           /                   \
    CA ----+--------------------+---- CP
          |                     |
          |   PICK ONLY TWO     |
          |                     |
    AP ----+--------------------+
               Partition
               Tolerance (P)
```

| Property | Description |
|---|---|
| **C — Consistency** | Every read gets the most recent write or an error |
| **A — Availability** | Every request gets a (non-error) response |
| **P — Partition Tolerance** | System works despite network partitions |

### Database Categories:

```
CA Systems (no Partition Tolerance):
└── MySQL, PostgreSQL — Good for financial/transactional data (ACID)
    Use for: Order Management, Payment Service

CP Systems (Consistency + Partition Tolerance):
└── MongoDB, Redis — Strong consistency with some availability trade-off
    Use for: User session data, real-time leaderboards

AP Systems (Availability + Partition Tolerance):
└── Cassandra, Amazon DynamoDB — Eventual consistency, highly scalable
    Use for: Product catalog, user activity tracking, analytics
```

**Interview Tip:** "For microservices, P (partition tolerance) is non-negotiable in distributed systems. So the real choice is between CP and AP. Prefer AP systems with eventual consistency for most microservices, and CP/CA systems for strictly transactional operations like payments."

---

## 4. 12 Factor App

A methodology for building cloud-native web applications. Each factor is critical for microservices:

### Factor 1 — Codebase
- One codebase per service, multiple deploys (dev/stage/prod are different deployments, not different repos)

### Factor 2 — Dependencies
- Explicitly declare all dependencies in `pom.xml` or `build.gradle`
- Never rely on pre-installed software on the host

### Factor 3 — Config
- Store all environment-specific config (passwords, URLs) in **environment variables**, never in code
- Spring Cloud Config Server enables centralized config management

### Factor 4 — Backing Services
- Treat all external services (databases, queues, caches) as **attached resources**
- Never hardcode URLs — use Ribbon/Eureka for dynamic resolution

```
// BAD — hardcoded URL:
restTemplate.getForObject("http://localhost:8090/products", Product[].class);

// GOOD — named service alias:
restTemplate.getForObject("http://product-service/products", Product[].class);
// 'product-service' resolves dynamically via Eureka
```

### Factor 5 — Build, Release, Run
- **Build:** Compile code into a binary
- **Release:** Combine binary with environment config → unique immutable release
- **Run:** Execute the release in the environment
- Rule: No code changes at runtime (no editing JARs in production!)

### Factor 6 — Processes
- Execute app as **stateless processes**
- No in-memory session state, no local file storage
- Use distributed cache (Redis) for shared state

```
// BAD — storing user session in memory:
map<userId, sessionData> inMemoryCache;  // Dies when service restarts

// GOOD — distributed cache:
redis.set("session:" + userId, sessionData, TTL_30_MINUTES);
```

### Factor 7 — Port Binding
- App is self-contained; exposes service via port binding
- Spring Boot embeds Tomcat/Jetty — no external servlet container needed

### Factor 8 — Concurrency
- Scale out via multiple processes (horizontal scaling), not by making one process bigger
- Use Kubernetes/Docker to spin up multiple containers

### Factor 9 — Disposability
- Fast startup, graceful shutdown
- Services can be started/stopped at any time
- Ideal startup time: a few seconds

### Factor 10 — Dev/Prod Parity
- Keep development, staging, production **as similar as possible**
- Use Docker to ensure identical runtime environments

### Factor 11 — Logs
- Treat logs as event streams; output to stdout
- Centralized log aggregation (ELK stack, Splunk) handles persistence

### Factor 12 — Admin Processes
- Run admin/management tasks as one-off processes
- Example: database migrations run as separate process, not bundled in app startup

**Interview Tip:** "The 12 Factor App is the blueprint for cloud-native microservices. Stateless processes (Factor 6), externalized config (Factor 3), and disposability (Factor 9) are the most commonly tested principles in design interviews."

---

## 5. Introduction to Microservices

### Definition

> "Microservices are small, autonomous services that work together." — Sam Newman
>
> "Loosely coupled service-oriented architecture with bounded contexts." — Adrian Cockcroft
>
> "A microservice architecture is the natural consequence of applying SRP at the architectural level." — Toby Clemson

### Architecture Overview

```
                    [Client: Browser / Mobile / IoT]
                                 |
                        [API Gateway / Zuul]
                       /    |        |     \
                      /     |        |      \
             [User    ] [Order ] [Payment] [Inventory]
             [Service ] [Svc   ] [Service] [Service  ]
                 |          |         |         |
               [DB]       [DB]      [DB]       [DB]
             (MySQL)    (Postgres) (Redis)  (Cassandra)
```

### 10 Key Characteristics

1. **High Cohesion** — one business capability per service
2. **Loose Coupling** — autonomous, independently deployable
3. **Bounded Context** — service owns its domain data
4. **Organized by Business Capability** (not technical layers)
5. **Continuous Delivery** — automated pipelines
6. **Versioning** — backward-compatible API versions
7. **Fault Tolerance** — one failure doesn't cascade
8. **Decentralized Data Management** — each service owns its DB
9. **Eventual Consistency** — async, event-driven updates
10. **Stateless Security** — JWT + OAuth2

### Benefits

| Benefit | Explanation |
|---|---|
| Independent Deployment | Teams deploy without coordination |
| Technology Diversity | Each service can use different language/DB |
| Selective Scalability | Scale only the high-traffic service |
| Resilience | One service down ≠ whole system down |
| Smaller Codebases | Easier to understand, maintain, onboard |

### Challenges

| Challenge | Solution |
|---|---|
| DevOps complexity | Kubernetes, Docker, CI/CD pipelines |
| Distributed computing complexity | Circuit Breakers, Retries, Async messaging |
| Configuration management | Spring Cloud Config Server |
| Multi-version routing | API Gateway + Canary Releasing |
| Log aggregation | Distributed tracing (Sleuth + Zipkin) |
| Service discovery | Eureka / Consul |

---

## 6. Domain Driven Design & Bounded Context

### Domain Driven Design (DDD)

DDD is a **modelling technique** for decomposing complex domains. Coined by Eric Evans.

**Key Principles:**
1. Focus on the **core domain** and domain logic
2. Base designs on **models of the domain**
3. Creative collaboration between technical and domain experts

### Bounded Context

**Definition:** A boundary within which a domain model is defined and applicable. Each bounded context has its own model that is **opaque to other contexts**.

**The Problem with a Unified Model:**

```
// BAD — One "Customer" class shared across ALL services:
class Customer {
    int id;
    string name;
    Address billingAddress;     // Only Payment needs this
    Address shippingAddress;    // Only Shipment needs this
    string mobile;              // Only Notification needs this
    string email;               // Only Notification needs this
    string loyaltyPoints;       // Only CRM needs this
    // This model is too complex and creates tight coupling!
}
```

**The Bounded Context Solution:**

```cpp
// Payment Service's Customer (its own bounded context):
struct CustomerInPayment {
    string id;
    string name;
    Address billingAddress;  // Only what Payment needs
};

// Shipment Service's Customer:
struct CustomerInShipment {
    string id;
    string name;
    string mobile;
    Address shippingAddress;  // Only what Shipment needs
};

// Notification Service's Customer:
struct CustomerInNotification {
    string id;
    string name;
    string mobile;
    string email;  // Only what Notification needs
};
```

### How to Partition Microservices

Apply Bounded Context to an e-commerce system:

```
eShop Domain
├── Product Catalog Service   → search, filter, product info
├── Inventory Service         → stock levels, availability
├── Order Service             → create/cancel orders
├── Payment Service           → process payments, refunds
├── Shipment Service          → track packages, delivery
├── User Account Service      → registration, preferences
├── Notification Service      → emails, SMS
├── Recommendation Service    → ML-based suggestions
├── Demand Generation         → marketing campaigns
└── Reviews & Ratings         → user feedback
```

**How Big Should a Microservice Be?**

> "As small as possible but as big as necessary to represent the domain concept they own." — Martin Fowler

Rule: Use **Bounded Context + SRP** as the sizing criterion, not lines of code.

**Amazon's Rule:** Each service is owned by a "two-pizza team" (team small enough that two pizzas can feed them).

---

## 7. Polyglot Persistence

**Concept:** Use different databases for different services based on their individual needs.

```
eShop System — Polyglot Persistence:

Order Service      → MySQL / PostgreSQL  (ACID transactions needed)
Product Catalog    → MongoDB             (flexible schema, documents)
User Activity      → Cassandra / DynamoDB (high write throughput, key-value)
Session Cache      → Redis               (in-memory, fast lookup)
Social Graph       → Neo4j               (relationships, recommendations)
Search             → Elasticsearch       (full-text search)
```

**When to choose which DB:**

| Database Type | Use When | Examples |
|---|---|---|
| RDBMS | Financial data, transactional integrity needed | MySQL, PostgreSQL |
| Document DB | Flexible schema, hierarchical data | MongoDB |
| Key-Value | Fast cache, session storage | Redis, DynamoDB |
| Wide-Column | Time-series, analytics, high write throughput | Cassandra, HBase |
| Graph DB | Social networks, recommendations | Neo4j |

**Can Polyglot Persistence be used in Monoliths?** Yes! But in microservices it's essential because each service *must* own its data store.

---

## 8. Microservices vs Monolith vs SOA

### Microservices vs Monolith

| Aspect | Monolith | Microservices |
|---|---|---|
| Deployment unit | Single WAR/EAR | Dozens of independent containers |
| Scaling | Scale the whole app | Scale only the bottleneck service |
| Technology | Single stack | Polyglot (Java, Go, Python, etc.) |
| Team structure | Single large team | Autonomous 2-pizza teams |
| Failure impact | One bug can bring entire app down | Isolated to one service |
| Communication | In-process method calls | Network (REST, AMQP) |
| Database | Shared RDBMS | Private per-service |
| Development speed | Slower (regression risk) | Faster (independent releases) |

```
Monolith:
[Browser] → [War File (UI + BL + DB Layer)] → [Shared MySQL DB]
              All in one JVM process

Microservices:
[Browser] → [API Gateway] → [Order Svc]    → [Order DB]
                          → [Product Svc]  → [Product DB]
                          → [Payment Svc]  → [Payment DB]
              Each is an independent JVM + Docker container
```

**Is in-process communication faster than network calls?**
Yes, but it's a trade-off worth making:
- Modern VPC optical fibre: petabyte/second, millions of API calls/second
- Microservices scalability advantages far outweigh the network overhead
- A monolith may be faster initially, but doesn't scale horizontally

### Microservices vs SOA

| Aspect | SOA | Microservices |
|---|---|---|
| Communication | SOAP over ESB (fat middleware) | REST over HTTP, AMQP (dumb pipes) |
| Deployment | Often deployment monoliths | Fully automated, independent deployment |
| Service Size | Larger, coarser-grained | Smaller, fine-grained |
| Data | Often shared databases | Each service owns its DB |
| Technology | Platform-driven, vendor lock-in | Technology agnostic |
| Guidance | No clear partitioning strategy | Bounded Context, DDD |

**Key insight:** Microservices = SOA done well. They are an **evolution**, not a replacement.

> "The value of the term Microservices is that it allows us to put a label on a useful subset of the SOA terminology." — Martin Fowler

**Small services vs Microservices:** Not all small services are microservices. A service must follow Bounded Context and SRP to qualify — size alone isn't the criterion.

---

## 9. Communication Patterns

### Synchronous vs Asynchronous

```
SYNCHRONOUS:
Client → Service A → (blocks, waits) → Service B → response
[Thread A blocked for entire duration]
Risk: If Service B is slow/down, Service A times out and the thread is wasted

ASYNCHRONOUS:
Client → Service A → publishes event to Queue → returns immediately
                     Queue → Service B → processes at its own pace
[No blocking; higher throughput on same hardware]
```

**When to use each:**

```
USE SYNCHRONOUS when:
- GET requests (read-only, aggregation at API Gateway)
- Response is needed immediately (user-facing query)
- Real-time validation

USE ASYNCHRONOUS when:
- POST/PUT/DELETE (anything that modifies data)
- Cross-service notifications (order created → notify, update inventory, ship)
- Batch processing, reporting
- Decoupled workflows (choreography)
```

**Final Advice from the book:**
1. Use async (RabbitMQ/AMQP) for all data-modifying operations
2. Use sync only for GET aggregation at API Gateway (no business logic transformations)
3. If services still need sync GET calls between them → rethink the partitioning

---

### Orchestration vs Choreography

**Orchestration (Anti-pattern):**

```
Central Orchestrator decides what happens, calls each service:

RecommendationService (orchestrator)
    ├── calls OrderService.getHistory(userId)    [sync]
    ├── calls ProductService.getDetails(items)  [sync]
    └── calculates recommendations

Problems:
- Tight coupling (Recommendation depends on Order & Product service APIs)
- Orchestrator becomes a bottleneck
- Hard to scale independently
```

**Choreography (Preferred):**

```
Event-driven; each service reacts to events:

OrderService → publishes [ORDER_CREATED event] to Message Queue
    ↓
RecommendationService subscribes → builds recommendation from event data
NotificationService subscribes → sends order confirmation email
InventoryService subscribes → adjusts stock

Benefits:
- Loose coupling (services don't know about each other)
- Highly scalable
- Easy to add new consumers without changing OrderService
```

```cpp
// Pseudocode: Choreography with event publishing
class OrderService {
public:
    void createOrder(Order order) {
        orderRepository.save(order);
        
        // Publish event — don't call other services directly
        Event event = {
            .type = "ORDER_CREATED",
            .data = order.toJson()
            // Note: Event carries DATA, not ACTIONS
        };
        messageQueue.publish("order.events", event);
    }
};

class NotificationService {
public:
    void onOrderCreated(Event event) {   // Subscribed to "order.events"
        Order order = parseOrder(event.data);
        emailService.sendConfirmation(order.customerId, order.id);
    }
};
```

---

## 10. ACID & Distributed Transactions

### ACID Properties

| Property | Definition |
|---|---|
| **A — Atomicity** | All-or-nothing; either all operations commit or all rollback |
| **C — Consistency** | DB moves from one valid state to another; constraints satisfied |
| **I — Isolation** | In-progress transactions are invisible to each other |
| **D — Durability** | Committed data survives crashes |

### The Problem in Microservices

```
Scenario: Create an Order and deduct Inventory atomically

Order Service DB (MySQL) ←───────────────────────┐
Inventory Service DB (MongoDB) ←─────────────────┘
          ↑ Both must succeed or both must fail
          ↑ But they are separate databases!
```

### Solution 1: Two-Phase Commit (2PC) — AVOID

```
Phase 1 (Prepare):
Coordinator → OrderDB: "Prepare to commit"
Coordinator → InventoryDB: "Prepare to commit"
Both respond: "Ready"

Phase 2 (Commit):
Coordinator → OrderDB: "Commit"
Coordinator → InventoryDB: "Commit"

Problems:
- Blocking protocol (locks held during coordination)
- Coordinator becomes single point of failure
- Network partitions leave DBs in inconsistent state
- Fragile and complex — NOT recommended for microservices
```

### Solution 2: Eventual Consistency (RECOMMENDED)

```cpp
// Pseudocode: Saga pattern with compensating transactions
class OrderSaga {
    void execute(Order order) {
        // Step 1: Reserve Inventory
        bool inventoryReserved = inventoryService.reserve(order.items);
        if (!inventoryReserved) {
            // Compensation: nothing to undo yet
            return FAILED;
        }
        
        // Step 2: Charge Payment
        bool paymentSucceeded = paymentService.charge(order.amount);
        if (!paymentSucceeded) {
            // Compensation: release inventory
            inventoryService.release(order.items);
            return FAILED;
        }
        
        // Step 3: Confirm Order
        orderRepository.save(order);
        // Eventually consistent — may take milliseconds to propagate
        return SUCCESS;
    }
};
```

**Transactional Outbox Pattern (Atomically publish event + DB write):**

```cpp
// Problem: Write to DB and publish to queue — one might fail

// Solution: Write event to local DB table in same transaction
void createOrder(Order order) {
    transaction.begin();
    
    orderRepository.save(order);  // Write order
    
    // Write event to outbox table in SAME local transaction
    OutboxEvent event = { "ORDER_CREATED", order.toJson() };
    outboxRepository.save(event);  // Same DB, same transaction!
    
    transaction.commit();  // Atomic: both or neither
    
    // Separate process reads outbox and publishes to message broker
    // If publish fails, it can retry from outbox
}
```

---

## 11. Deployment Strategies

### Continuous Integration (CI)

Every developer commit triggers an automated pipeline:

```
Git Commit → Jenkins Pipeline:
  Step 1: Compile & Build
  Step 2: Unit Tests
  Step 3: Integration Tests  
  Step 4: Static Analysis (FindBugs, PMD)
  Step 5: Code Coverage (JaCoCo)
  Step 6: End-to-End Tests (Selenium)
  Step 7: Deploy to Staging
```

### Continuous Delivery (CD)

Any passing pipeline can be deployed to production **at any time**:

```
Amazon: deploys every 11.6 seconds (2011 stat)
GitHub: deploys ~60 times per day
Facebook: 2 deploys per day
Etsy: 50+ deploys per day
```

### Zero-Downtime Deployment — Blue/Green

```
BEFORE DEPLOYMENT:
[Load Balancer] → [Blue v1.0 ✅ LIVE] → DB

DEPLOYMENT IN PROGRESS:
[Load Balancer] → [Blue v1.0 ✅ LIVE] → DB
                  [Green v2.0 🔄 Staging]

AFTER SMOKE TESTS PASS:
[Load Balancer] → [Green v2.0 ✅ LIVE] → DB
                  [Blue v1.0 ⏸ Standby for rollback]

ROLLBACK (if needed):
[Load Balancer] → [Blue v1.0 ✅ LIVE] (instant, just switch traffic)
```

**Blue/Green with Database Changes:**

| DB Change Type | Strategy |
|---|---|
| Backward-compatible (new column) | Add column as nullable; both v1 & v2 work during transition |
| Breaking change (rename column) | Create new column → copy data → deploy v2 → drop old column |

### Canary Releasing

```
Slower, safer alternative to Blue/Green:

Phase 1: Route 5% of traffic to v2.0
         95% → v1.0, 5% → v2.0

Phase 2: Monitor; if OK, route 20% to v2.0
         80% → v1.0, 20% → v2.0

Phase 3: Gradually increase to 100%
         100% → v2.0

API Gateway routes selected endpoints to new service.
Both versions coexist for longer period.
```

### Strangulation Pattern

Incrementally decompose a monolith into microservices:

```
Phase 1: Monolith handles everything
[API Gateway] → [Legacy Monolith]

Phase 2: Extract /first/** to new microservice
[API Gateway] → /first/**  → [New Microservice v1]
              → /**        → [Legacy Monolith]

Phase 3: Keep migrating endpoints one by one
[API Gateway] → /first/**    → [New Microservice]
              → /second/**   → [New Microservice 2]
              → /**          → [Legacy Monolith] (shrinking)
```

---

## 12. Service Discovery — Eureka

### Why Service Discovery?

In cloud environments, services come and go dynamically. You can't hardcode IPs.

```
Static (BAD):
ServiceA → always calls http://192.168.1.5:8080/products
           What if the server moves? Port changes? New instances added?

Dynamic (GOOD — Eureka):
ServiceA → looks up "product-service" in Eureka registry
           Eureka returns: [192.168.1.5:8080, 192.168.1.6:8081, ...]
           Load balance across available instances
```

### How Eureka Works

```
Architecture:

[Eureka Server] ← registry of all services

[Service A] ──── registers at startup ──────→ [Eureka Server]
              ──── heartbeat every 30s ──────→
              ←─── evicted if no heartbeat in 90s ──

[Service B] ──── fetches registry every 30s ─→ [Eureka Server]
              ──── caches registry locally ──
              ──── uses Ribbon to call A ──────→ [Service A]
```

**Key Behaviors:**
- Services register with Eureka on startup
- Heartbeat every 30 seconds; removed from registry after ~90s of no heartbeat
- Clients cache the registry locally → can work even if Eureka goes down
- One Eureka cluster per region (us, asia, europe); replicated across nodes

**Zone-Aware Routing:** Services prefer other services in the same zone to minimize latency:

```yaml
# Service in Zone 1 prefers other Zone 1 services:
eureka.instance.metadataMap.zone: zone1
eureka.client.preferSameZoneEureka: true
```

**Consul** is an alternative to Eureka (REST-based, supports health checks).

---

## 13. API Gateway

### What is an API Gateway?

A single entry point for all client requests. Routes to appropriate microservices and provides cross-cutting concerns.

```
Clients → API Gateway → Routes to microservices
              ↓
          Cross-cutting concerns:
          - Authentication / Authorization
          - Rate Limiting
          - SSL Termination
          - Request Aggregation
          - Protocol Translation (HTTP → AMQP)
          - Circuit Breaking (Hystrix)
          - Load Balancing (Ribbon)
          - Request/Response Logging
          - Header Stripping (hide internal tokens)
```

### API Gateway in Practice (Zuul)

```yaml
# zuul routes configuration:
zuul:
  routes:
    customer-service:
      path: /customer/**
      serviceId: customer-service   # resolves via Eureka
    product-service:
      path: /product/**
      serviceId: product-service
  # Protect internal endpoints from leaking:
  ignored-patterns: /*/health, /*/internal
  # Strip sensitive headers before forwarding:
  sensitiveHeaders: Authorization, X-Internal-Token
```

**Request Aggregation Pattern:**

```cpp
// API Gateway aggregates responses from multiple services:
// Avoids multiple round-trips from mobile client
class ProductPageAggregator {
    ProductPageResponse getProductPage(string productId) {
        // Parallel calls to multiple services:
        auto productFuture = productService.getDetails(productId);
        auto reviewsFuture = reviewService.getReviews(productId);
        auto inventoryFuture = inventoryService.getStock(productId);
        
        // Wait for all and aggregate:
        return {
            .product = productFuture.get(),
            .reviews = reviewsFuture.get(),
            .inventory = inventoryFuture.get()
        };
        // Single response to client — saves mobile bandwidth
    }
};
```

---

## 14. Client-Side Load Balancing — Ribbon

### How Ribbon Works

```
Traditional (server-side) load balancing:
Client → [HAProxy/Nginx Load Balancer] → [Server 1, 2, 3]

Ribbon (client-side) load balancing:
Client (with Ribbon) → fetches server list from Eureka
                     → picks one using load-balancing algorithm
                     → calls server directly (no intermediary)
```

**Ribbon's 3 Internal Components:**

| Component | Purpose |
|---|---|
| **Rule** | Determines which server to pick (default: Round Robin) |
| **Ping** | Background thread checking server liveness |
| **ServerList** | Static or dynamic list of servers (from Eureka) |

**Available Load Balancing Algorithms:**

```
RoundRobinRule         → Distribute equally, circular order (default)
WeightedResponseTimeRule → Prefer servers with faster response time
AvailabilityFilteringRule → Skip servers with open circuit breakers
ZoneAwareLoadBalancer  → Prefer servers in same zone
```

**Retry on Failure:**

```yaml
# Retry failed requests on next available server:
ribbon:
  MaxAutoRetries: 3          # Retries on same server
  MaxAutoRetriesNextServer: 1 # Tries on next server if all retries fail
  OkToRetryOnAllOperations: true
```

---

## 15. Circuit Breaker Pattern & Hystrix

### The Problem

```
Normal operation:
ServiceA → ServiceB → response ✅

ServiceB becomes slow/unresponsive:
ServiceA → ServiceB (hangs for 30s timeout)
Thread in ServiceA is BLOCKED for 30 seconds
High traffic → ALL threads blocked → ServiceA becomes unresponsive too
→ Cascading failure brings down the entire system
```

### Circuit Breaker States

```
         Failure count < threshold         Failure count ≥ threshold
CLOSED ──────────────────────────────────→ OPEN
(all calls pass through)                   (all calls FAIL immediately,
                                            no actual call made)
         ↑                                        ↓
         │                                   After timeout period
         │                                   (default: 5 seconds)
         └─────────────── HALF-OPEN ←────────────┘
                         (test call goes through;
                          if succeeds → CLOSED;
                          if fails → OPEN again)
```

**Conditions that OPEN the circuit:**
- Exception thrown (HTTP 500, connection refused)
- Call takes longer than timeout (default: 1 second)
- Thread pool exhausted (too many concurrent requests)

### Circuit Breaker vs Try/Catch

| Aspect | Try/Catch | Circuit Breaker |
|---|---|---|
| Prevents network calls after threshold? | No (still tries every time) | Yes (fails fast) |
| Fallback execution | Manual boilerplate | Built-in |
| Thread pool isolation | No | Yes (Bulkhead) |
| Metrics/monitoring | No | Yes (Hystrix Dashboard) |
| Auto recovery | No | Yes (Half-Open state) |

### Hystrix

Netflix's implementation of circuit breaker pattern.

**Key Features:**
1. **Latency & Fault Tolerance** — stop cascading failures, graceful fallback
2. **Real-time Metrics** — traffic volume, error %, latency percentiles, success/failure counts
3. **Request Collapsing** — batch multiple concurrent requests into a single backend call
4. **Bulkhead Pattern** — isolate thread pools per dependency

```java
// Pseudocode equivalent in C++:
class BookService {
public:
    // @HystrixCommand(fallbackMethod = "getCachedBooks")
    vector<Book> getRecommendedBooks() {
        // Circuit breaker wraps this call:
        return httpClient.get("http://books-service/recommended");
        // If this fails N times → circuit OPENS → fallback called
    }
    
    // Fallback method — same signature:
    vector<Book> getCachedBooks() {
        return cache.get("recommended_books");  // stale but available
        // Graceful degradation — user sees old data, not an error page
    }
};
```

**Hystrix Metrics (all aggregated by Turbine):**
- Traffic volume per second
- Error percentage
- Latency percentiles (p50, p95, p99)
- Number of open circuits
- Thread pool utilization

---

## 16. Bulkhead Design Pattern

**Origin:** Ship hulls divided into watertight compartments — damage to one doesn't sink the ship.

### The Problem Without Bulkhead

```
WebFront (30 total threads):
  ├── Product Catalog Service calls → uses threads
  ├── Reviews Service calls → uses threads  ← Reviews Service hangs
  └── Order Service calls → uses threads

Reviews Service hangs → ALL 30 threads eventually waiting on it
→ WebFront has NO threads to serve Product/Order requests
→ Entire WebFront goes down because of ONE bad service
```

### Bulkhead Solution

```
WebFront with Hystrix Bulkhead:
  ├── Product Catalog thread pool (max 10 threads)
  ├── Reviews thread pool (max 10 threads)  ← Limited to 10
  └── Order Service thread pool (max 10 threads)

Reviews Service hangs → only 10 threads tied up
→ Other 20 threads still serve Product & Order requests
→ System degrades gracefully instead of crashing
```

### Two Bulkhead Implementations in Hystrix

**Thread Isolation (Default — Recommended):**

```cpp
// Each command group gets its own thread pool:
struct HystrixThreadPoolConfig {
    int coreSize = 10;      // Max concurrent calls
    int maxQueueSize = -1;  // No queue (reject immediately if pool full)
};
// Advantage: Can timeout the call (thread can be interrupted)
// Disadvantage: Thread context switching overhead
```

**Semaphore Isolation:**

```cpp
// Uses a semaphore to limit concurrent calls:
struct HystrixSemaphoreConfig {
    int maxConcurrentRequests = 10;
};
// Advantage: No thread overhead; better for security context propagation
// Disadvantage: Cannot timeout (semaphore can't interrupt calling thread)
// Use when: Propagating OAuth AccessToken to downstream service
```

---

## 17. Externalized Configuration

### Why Externalize Config?

```
Problem: Config hardcoded in code:
String dbUrl = "jdbc:mysql://prod-db:3306/orders";  // Committed to Git!
// ❌ Password in source code
// ❌ Different URL for dev/stage/prod requires code change
// ❌ Config change requires redeployment

Solution: Spring Cloud Config Server
Config Server → reads from Git repo → serves to all microservices
Each microservice → fetches config at startup from Config Server
```

### Config Server Architecture

```
Git Repos:
  config-dev  (dev environment configs)
  config-qa   (QA environment configs)
  config-prod (production configs — highly secured)

                Config Server
               /      |      \
              /       |       \
[Service A]  [Service B]  [Service C]
    ↑ each fetches its own config at startup
```

### bootstrap.yml vs application.yml

| File | Purpose | Loaded When |
|---|---|---|
| `bootstrap.yml` | Spring Cloud config; specifies where to find Config Server | Very first, before app context |
| `application.yml` | Application-specific defaults | After bootstrap completes |

```yaml
# bootstrap.yml — specifies Config Server location:
spring:
  application:
    name: order-service
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true  # Fail startup if Config Server unreachable

# application.yml — local defaults (overridden by Config Server):
server:
  port: 8080
spring:
  datasource:
    url: jdbc:h2:mem:testdb  # Default; Config Server overrides for prod
```

### Config-First vs Discovery-First Bootstrap

```
Config-First (default):
Service starts → contacts Config Server directly (knows its URL)
  └── Simpler; Config Server must start first

Discovery-First:
Service starts → contacts Eureka → finds Config Server dynamically
  └── Config Server URL not hardcoded
  └── Extra network round-trip at startup
  └── Enable with: spring.cloud.config.discovery.enabled: true
```

### Live Config Refresh

```java
// @RefreshScope beans reload config without restart:
@RefreshScope
@RestController
class FeatureController {
    @Value("${feature.newCheckout.enabled}")
    private boolean newCheckoutEnabled;
    
    @GetMapping("/checkout")
    public String checkout() {
        if (newCheckoutEnabled) {
            return "New checkout flow";
        }
        return "Old checkout flow";
    }
}
// POST /actuator/refresh → config reloaded, bean re-created
```

---

## 18. Security — OAuth2, JWT

### Why Not Basic Authentication in Microservices?

| Problem | Explanation |
|---|---|
| Credentials required every request | Password sent every time (even over HTTPS, risky) |
| No user/machine distinction | Can't differentiate human users from service accounts |
| No scopes or token extension | Can't carry additional claims (roles, permissions) |
| BCrypt performance cost | 4-8 password hashes/sec on small machines; every request pays this cost |
| Mobile credential storage | Storing plaintext password on device is dangerous |

### OAuth2 Framework

```
Four Roles:
1. Resource Owner    = End User (or the service that owns the data)
2. Resource Server   = Microservice (holds protected data)
3. Authorization Server = Issues access tokens (after verifying identity)
4. Client            = App making the request (mobile app, web app, microservice)
```

### OAuth2 Grant Types

| Grant Type | When to Use | Example |
|---|---|---|
| **Authorization Code** | Web apps; client secret kept server-side | Login with Google on a web app |
| **Resource Owner Password** | Trusted clients (first-party mobile apps) | Your own company's Android app |
| **Client Credentials** | Service-to-service (no user involved) | Scheduled batch jobs, inter-service calls |
| **Implicit** | Single-page apps where client secret can't be hidden | Angular/React apps |
| **Refresh Token** | Getting new access token after expiry | All grant types except Client Credentials |

### JWT — JSON Web Token

**Structure:** `header.payload.signature` (all Base64 encoded)

```
HEADER:
{
  "alg": "HS256",    // HMAC-SHA256 signing algorithm
  "typ": "JWT"
}

PAYLOAD (Claims):
{
  "uid": "85923dde-...",
  "user_name": "user@example.com",
  "scope": ["read", "write"],
  "exp": 1520017228,          // Expiration timestamp
  "authorities": ["ROLE_USER", "ROLE_ADMIN"],
  "client_id": "mobile-app"
}

SIGNATURE:
HMACSHA256(base64(header) + "." + base64(payload), secret_key)
```

**How JWT enables stateless security:**

```cpp
// No database lookup needed to validate JWT:
bool validateJWT(string token, string publicKey) {
    // 1. Decode header and payload from Base64
    Header header = decodeBase64(token.split(".")[0]);
    Payload payload = decodeBase64(token.split(".")[1]);
    string signature = token.split(".")[2];
    
    // 2. Verify signature with public key (cryptographically)
    string expectedSig = HMACSHA256(header + "." + payload, publicKey);
    if (signature != expectedSig) return false;  // Tampered!
    
    // 3. Check expiry
    if (payload.exp < currentTime()) return false;  // Expired!
    
    return true;  // Valid — no DB lookup needed!
}
```

**AccessToken vs RefreshToken:**

```
AccessToken:
- Short-lived (minutes to hours)
- Used in every API request: Authorization: Bearer <token>
- Contains user claims (roles, permissions)

RefreshToken:
- Long-lived (days to months)
- Used ONLY to get a new AccessToken when current one expires
- Not sent with every API request
- Requires client credentials to use
```

**Token Relay (Service-to-Service):**

```
User logs in → gets AccessToken from Auth Server
User calls API Gateway with AccessToken
API Gateway forwards request to Service A with same AccessToken
Service A calls Service B — relays user's AccessToken
Service B validates token — knows who the user is

This preserves user security context across service calls.
```

**JWT Security — Revoking Tokens:**

```
Problem: JWT is stateless; can't be "unrevoked" before expiry

Solutions by scope of breach:
- System-wide breach   → Rotate the private signing key (all tokens invalidated)
- Client-wide breach   → Rotate client credentials (refreshTokens become invalid)
- Single user breach   → Track device IDs; block the specific device

Short-lived access tokens (15min) + refresh tokens = best practice
```

**Implementing Logout with JWT:**

```
Option 1: Clear tokens from client storage (simplest, but token still valid server-side until expiry)
Option 2: Short-lived AccessTokens + RefreshTokens (token expires quickly anyway)
Option 3: Maintain a server-side token blacklist (breaks statelessness but enables true revocation)
```

---

## 19. Best Practices & Anti-Patterns

### Best Practices

| Practice | Description |
|---|---|
| **Partition by Business Capability** | Use DDD Bounded Context, not technical layers |
| **Stateless Operations** | No sticky sessions, no local cache |
| **Design for Failure** | Circuit breakers, retries, bulkheads |
| **Async Communication** | Prefer AMQP/Kafka for inter-service messaging |
| **Eventual Consistency** | Accept that data syncs asynchronously |
| **Idempotent Operations** | Include unique request IDs to handle retries safely |
| **Versioned APIs** | `/api/v1/`, `/api/v2/` — don't break old consumers |
| **Share as Little as Possible** | No unified domain models across services |
| **DevOps Culture** | Fully automate CI/CD pipelines |

### API Versioning

```
URL Versioning (most common):
  GET https://api.example.com/v1/orders
  GET https://api.example.com/v2/orders

Header Versioning:
  GET https://api.example.com/orders
  Header: API-Version: 2
```

**Rule:** Any non-backward-compatible change must result in a new version.

### Sharing Code — What to Share vs Not Share

```
✅ SHARE (as versioned JAR in private Maven repo):
- ServiceResponse<T> wrapper class
- Custom RestTemplate with TLS/retry
- Logging utilities
- Authentication client libraries

❌ DO NOT SHARE (domain models):
// Instead of one Customer model shared by all services:
// Each service defines its OWN Customer with only its needed fields

// Payment Service:
struct Customer { string id, name; Address billingAddress; };

// Shipment Service:  
struct Customer { string id, name, mobile; Address shippingAddress; };
```

**Why not share domain models?**
- Violates Bounded Context
- Creates tight coupling (changing one field affects all services)
- Models grow complex and hard to understand
- In microservices, **violating DRY for domain models is correct**

### Reporting Microservice — Right Approach

```
WRONG — HTTP Pull (naive):
Reporting Service → GET /orders (from Order Service)
                 → GET /inventory (from Inventory Service)
                 → GET /payments (from Payment Service)
Problems: Inefficient for bulk data; loads operational DBs

WRONG — Direct DB Pull:
Reporting Service → reads Order DB directly
Problems: Tight coupling; schema changes break reporting

CORRECT — Async Event-Driven Push (Data Pump pattern):
Order Service    → publishes [ORDER_CREATED events] ──→ Reporting DB
Inventory Service → publishes [INVENTORY_UPDATED events] ─→ Reporting DB
Payment Service  → publishes [PAYMENT_PROCESSED events] ──→ Reporting DB

Reporting Service → reads its OWN reporting-optimized DB
                  → queries are denormalized, fast, read-only
```

### Smart Endpoints & Dumb Pipes

Martin Fowler's microservices characteristic:

```
Unix philosophy:
ps elf | grep java
   ↑         ↑
(smart      (dumb pipe: just
endpoint)    routes output)

SOA (BAD): ESB (fat, smart pipe) with business logic in the middleware
Microservices (GOOD): Simple message queues (RabbitMQ) that just route messages.
Business logic belongs in the services (endpoints), NOT in the transport layer.
```

---

## 20. Testing Strategies

### Mike Cohn's Test Pyramid

```
              ╱╲
             ╱  ╲
            ╱ E2E╲       ← Few (expensive, slow, brittle)
           ╱──────╲
          ╱Contract╲     ← Some (CDC tests)
         ╱──────────╲
        ╱ Integration ╲  ← More (service-level)
       ╱────────────────╲
      ╱    Unit Tests    ╲  ← Most (fast, isolated, cheap)
     ╱────────────────────╲
```

**Rule:** More tests at the bottom, fewer at the top. Unit tests should be the largest layer.

---

### Unit Tests

**Focus:** Single component in isolation; all dependencies mocked.

```cpp
// Pseudocode: Unit testing a service
class OrderServiceTest {
    OrderService* subject;      // Class under test
    OrderRepository* mockRepo;  // Mocked dependency
    
    void setUp() {
        mockRepo = createMock<OrderRepository>();
        subject = new OrderService(mockRepo);  // Inject mock
    }
    
    void testCreateOrder_Success() {
        Order input = buildOrder("item123", 2);
        
        // Setup mock expectation:
        when(mockRepo.save(input)).thenReturn(input.withId("order-456"));
        
        Order result = subject.createOrder(input);
        
        // Verify:
        assertEquals("order-456", result.id);
        verify(mockRepo).save(input);  // Ensure save was called exactly once
    }
};
```

**Rules:**
- Do NOT load Spring Context in unit tests
- Mock ALL external dependencies
- Service layer and utility classes are ideal candidates
- Controller layer is better for integration tests

---

### Integration Tests

**Focus:** How your code interacts with real infrastructure (DB, filesystem, network).

```
Types of Integration Tests:
├── HTTP Integration Test    → Real HTTP hits your REST controller
├── Database Integration Test → Real DB queries (use H2 in-memory for speed)
├── FileSystem Test          → Real file system operations
└── External Service Test    → Real call to third-party service (or stub it)
```

**@DataJpaTest — Database Integration:**

```java
// Pseudocode equivalent:
@DataJpaTest  // Loads only JPA layer; auto-configures H2 in-memory DB
class ProductRepositoryTest {
    
    @Test
    void testFindByName() {
        // Arrange
        Product product = new Product("Widget", 9.99);
        entityManager.persist(product);  // Save to H2 in-memory DB
        
        // Act
        List<Product> found = productRepository.findByName("Widget");
        
        // Assert
        assertEquals(1, found.size());
        assertEquals("Widget", found.get(0).getName());
    }
    
    @After
    void tearDown() {
        productRepository.deleteAll();  // ← IMPORTANT: isolation!
    }
}
```

---

### Contract-Driven Tests (CDC Tests)

**Problem:** Service A calls Service B. How do we know Service B won't break Service A's expectations?

```
Traditional approach:
- Deploy both services → run integration test → detect breakage LATE

Consumer-Driven Contract (CDC) approach:
- Consumer (Service A) defines a contract (Pact JSON file)
  "I expect endpoint /api/orders/{id} to return { id, amount, status }"
- Provider (Service B) runs consumer's contract as a test
  "Does my API match what Service A expects?"
- Breakage detected EARLY in CI pipeline, before deployment
```

```
Pact Workflow:
1. Consumer writes contract: "I expect this request/response"
2. Pact generates contract file (pact.json)
3. Consumer shares pact.json with Provider team
4. Provider runs pact.json as a verification test
5. CI fails if Provider doesn't match Consumer's expectations
```

---

### End-to-End (E2E) Tests

**Focus:** Test the entire system as a black box through the public interface.

```
Selenium E2E Test:
1. Start entire Spring Application on random port
2. Open real Chrome browser
3. Navigate to application URL
4. Interact with UI (click buttons, fill forms)
5. Assert expected behavior

Coverage:
- User authentication flows
- Complete purchase workflows
- Error pages and validation
- Security (only authorized users access protected pages)
```

**Best Practices:**
- Keep E2E tests to a minimum (expensive, slow)
- Use Test Doubles / Pact for dependent services (don't spin up 100 services)
- Tests must be deterministic — avoid non-determinism (sleep(), uncleared DB state)

**Eliminating Non-Determinism:**

```
Causes of flaky tests:
1. Dependency on remote services → use Wiremock/stub instead
2. Async code with arbitrary timeouts → use polling loops
3. DB state pollution between tests → cleanup in @After / run in transactions

// Polling instead of sleep:
// BAD:
Thread.sleep(5000);  // Assume API call finishes in 5s (non-deterministic)

// GOOD:
int maxAttempts = 20;
while (maxAttempts-- > 0) {
    if (result.isReady()) break;
    Thread.sleep(100);  // Poll every 100ms, up to 2 seconds total
}
```

---

### Mock vs Stub

| | Stub | Mock |
|---|---|---|
| **Definition** | Pre-written fake implementation | Dynamically set up with expectations |
| **Written By** | Developer manually | Mocking framework (Mockito) |
| **Verifies Behavior** | No | Yes (verifies method was called) |
| **When to Use** | Simple fake responses | When you care about *how* a dependency was used |

```cpp
// STUB: Pre-written fake
class FakeOrderRepository : public OrderRepository {
public:
    Order findById(string id) override {
        return Order("test-order", 100.0);  // Always returns same thing
    }
};

// MOCK: Framework-generated, with expectations
auto mockRepo = createMock<OrderRepository>();
expect(mockRepo.findById("order-123")).toReturn(testOrder);
// + verification: mockRepo.findById was called exactly once
```

---

## 21. Performance & Caching

### Caching Strategies

```
1. SERVER-SIDE CACHE (distributed, transparent to clients):
   Service → Redis/Memcache (shared cache)
   All instances of a service see the same cached data

2. API GATEWAY CACHE:
   API Gateway caches frequent responses
   Reduces load on backend services
   Best for: static data, infrequently changing responses

3. CLIENT-SIDE CACHE:
   HTTP Cache-Control headers tell clients to cache
   ETags for conditional requests
   Eliminates repeated calls for same data
   Best for: product images, catalog data
```

### Thread Pool Tuning

```yaml
# Servlet container thread pool configuration:
server:
  tomcat:
    max-threads: 200       # Max concurrent requests
  undertow:
    io-threads: 4           # = CPU cores / 2
    worker-threads: 40      # = 10 * io-threads
  jetty:
    acceptors: 2            # = 1 + CPU cores / 16
    selectors: 8            # = CPU cores
```

**Rule of Thumb:**
- CPU-bound tasks: threads = CPU cores
- I/O-bound tasks (DB calls, network): threads = CPU cores × 10 (threads block waiting)

### HTTP/2 Protocol

Benefits over HTTP/1.1:
- **Header compression** — reduces payload size
- **Multiplexing** — multiple requests on single TCP connection
- **Server push** — server proactively sends resources client will need
- Reduces round-trips significantly for API-heavy applications

---

## 22. Monitoring, Logging & Tracing

### Distributed Tracing — Correlation IDs

**Problem:** A request fails. It spans 5 microservices. Where did it fail first?

```
Request enters API Gateway → assigns Correlation ID: "req-abc-123"
API Gateway → Service A (passes req-abc-123 in header)
Service A → Service B (passes req-abc-123 in header)
Service B → Service C (passes req-abc-123 in header)

Service C FAILS → logs "req-abc-123: Error: DB timeout"
Service B FAILS → logs "req-abc-123: Upstream error from C"
Service A FAILS → logs "req-abc-123: Upstream error from B"

→ Search all logs for "req-abc-123"
→ Find exact sequence of events across all services
→ Identify root cause: Service C's DB timeout
```

**Spring Cloud Sleuth** automatically adds correlation IDs (Trace ID, Span ID) to all log entries.
**Zipkin** visualizes distributed traces as timelines.

### Monitoring Stack

```
Metrics Collection:
  Each microservice → exposes /actuator/metrics endpoint
  Graphite → scrapes metrics, stores in time-series DB

Visualization:
  Grafana → queries Graphite → dashboards for all services
  Spring Boot Admin → health, JVM metrics, thread dumps, Hystrix stats

APM Tools:
  AppDynamics, Dynatrace → paid tools with advanced features

Circuit Breaker Metrics:
  Hystrix Dashboard → real-time stream of circuit breaker states
  Turbine → aggregates Hystrix streams from all services
```

---

## 23. Event Sourcing & CQRS

### Event Sourcing

Instead of storing current state, store every event that led to that state:

```cpp
// Traditional (store current state only):
struct Shipment {
    string id;
    string status;  // "PICKED_UP" — but we don't know history
    Location currentLocation;
};

// Event Sourcing (store every state change):
vector<ShipmentEvent> events = {
    { "ORDER_PLACED",    "2024-01-01 10:00", {0,0} },
    { "PICKED_UP",       "2024-01-01 14:00", {Warehouse} },
    { "IN_TRANSIT",      "2024-01-02 09:00", {Highway} },
    { "OUT_FOR_DELIVERY","2024-01-03 08:00", {City} },
    { "DELIVERED",       "2024-01-03 14:30", {Customer} }
};
// Current state = replay of all events
```

**Ideal Use Cases:** Shipping tracker, audit logs, financial ledgers — anywhere history matters.

**Warning:** Don't use Event Sourcing system-wide. It adds complexity. Use only for specific bounded contexts where full history is needed.

### CQRS — Command Query Responsibility Segregation

```
Separate READ and WRITE models:

WRITE side (Command):                    READ side (Query):
UpdateOrderStatus("order-123") ──────→   ReadOrder(123) 
        ↓                                     ↑
   Command Handler                      Query Handler
        ↓                                     ↑
   Write DB (normalized)                Read DB (denormalized, optimized for queries)
        ↓                                     ↑
   Publish event ──────────────────────────────┘
   (async sync)

Benefits:
- Write DB optimized for consistency
- Read DB optimized for query performance (can use separate DB type)
- Scale reads and writes independently
```

---
---

*Study Guide created from "All About Microservices" | Covers 100+ interview questions | Tailored for System Design Interviews*
