# System Design Architecture Document

  * **Project Name:** Flash Sale E Commerce Platform
  * **Team Name:** Code Catalyst
  * **Date:** 06/11/2025
  * **Prepared By:**
      * Janvi (2446384)
      * Deepak Katare (2446387)
      * Naman Dubey (2446386)
      * Suryaprakash Pandey (2446436)
  * **Reviewed By:** Divyang Panchasara

-----

# 1\. Introduction

## 1\.1 Purpose

The primary purpose of this system architecture design is to define a highly scalable, resilient, and performant E-commerce Platform specifically engineered to handle the extreme load and concurrency demands of Flash Sale events.
The architecture ensures:

  * **No Overselling:** Guaranteeing strong consistency in inventory management under massive concurrent demand.
  * **High Conversion Rates:** Minimizing latency across the critical purchase path to maximize the number of successful transactions.
  * **Fault Isolation:** Using a Microservices and Event-Driven Architecture (EDA) to prevent the failure of one service.

## 1\.2 Scope

The scope defines the boundaries and major components included in the system design.

  * Core E-commerce Functionality: Product browsing, user authentication, shopping cart management, order placement, and order history.
  * Flash Sale Specifics: Dedicated services and databases for managing sale rules, pre-caching inventory, atomic stock reduction, and enforcing per-user purchase limits.
  * Distributed System Infrastructure: API Gateway, Load Balancers, CDN, Caching Layers (Redis), Message Queues (Kafka), and container orchestration (Kubernetes).
  * Operational Readiness: Designs for observability (Logging, Monitoring, Tracing), CI/CD pipelines, and Infrastructure as Code (IaC).
  * Consistency & Security: Implementation of the Saga pattern for distributed transaction consistency and adherence to security standards (e.g., JWT, TLS, PCI compliance for payment integration).

## 1\.3 Overview

The E-commerce Flash Sale Platform is a high-performance, cloud-native application utilizing a **Microservices Architecture** built on **.NET** and deployed via **Kubernetes**.
The core design priority is maximizing transactional throughput and guaranteeing inventory consistency during extreme traffic spikes typical of flash sales.

**Key Architectural Pillars:**

  * **Frontend Access & Security:** User interaction occurs via mobile or web applications.
  * All external traffic is funnelled through an **API Gateway**, which handles **centralized request routing, security enforcement (JWT),** and **rate limiting**.
  * A **CDN** minimizes latency for static content delivery.
  * **Core Backend & Microservices:** The system is composed of specialized .NET microservices (e.g., **Inventory Service, Order Service, Flash Sale Service**).
  * Communication between these services is primarily handled through **asynchronous Domain Events** delivered via a **Message Queue (Kafka)**, ensuring high decoupling and resilience.
  * **Performance Layer:** A high-speed **Redis Cache** is strategically used on the critical path to hold **real-time flash sale stock counts** using atomic operations and managing user session data, which dramatically minimizes load on the persistent databases during peak traffic.

-----

## 2\. SYSTEM OVERVIEW

### 2.1 SYSTEM DESCRIPTION

The E-commerce Flash Sale Platform is a specialized, cloud-native application designed to extend standard e-commerce capabilities by offering high-volume, limited-time promotional sales with guaranteed performance and data consistency.
Built on a modern Microservices Architecture using the .NET framework and orchestrated by Kubernetes (K8s), the system prioritizes low-latency and high throughput to maximize conversion rates and prevent critical failures like overselling.

### 2.2 KEY FEATURES

#### 2.2.1 Performance and Scalability

  * **Atomic Inventory Management:** Uses Redis atomic operations (DECR) to handle real-time stock deductions on the critical path, ensuring zero overselling during massive concurrent purchase attempts.
  * **Horizontal Scaling:** Services are stateless and deployed on Kubernetes, allowing for dynamic scaling (via HPA) in response to instantaneous traffic spikes when a sale starts.
  * **Asynchronous Processing:** Leverages Kafka to move non-critical operations (notifications, analytics updates) off the main thread, dedicating synchronous resources to the purchase and payment flow.

#### 2.2.2 Resilience and Consistency

  * **Distributed Transaction Management:** Implements the Saga Pattern (orchestrated by the Order Service) to manage state across microservices.
  * If the Payment Service fails, compensation steps (like releasing inventory back to stock) are reliably executed.
  * **Circuit Breakers and Retries:** Integrated into service-to-service communication via .NET's HttpClientFactory and resilience libraries to isolate failures (e.g., if the Payment Gateway becomes slow or unresponsive).

#### 2.2.3 Core Flash Sale Functionality

  * **Sale Lifecycle Management:** Dedicated services for scheduling, starting, and ending flash sale events.
  * **User Guardrails:** Enforcement of strict **rate limiting** at the API Gateway and **purchase limits** (e.g., one item per user) in the Flash Sale Service to prevent bot abuse.
  * **Real-time Status:** Provides ultra-low-latency updates on product status and countdown timers.

-----

## 3\. SYSTEM ARCHITECTURE

A modern e-commerce platform requires a highly scalable, fault-tolerant, and performant architecture to handle varying traffic loads, especially during peak sales or promotional events.
A microservices-based approach with event-driven communication is ideal for achieving these goals.

### 3.1 High-Level Architecture Overview

Our e-commerce system will consist of several independent microservices, an API Gateway, various caching layers, robust data storage solutions, and external integrations, all orchestrated and monitored in a cloud environment.

  * **Clients:** Web browsers, mobile apps, and external systems (e.g., aggregators).
  * **CDN (Content Delivery Network):** For fast delivery of static assets (images, CSS, JS).
  * **Load Balancer:** Distributes incoming traffic across multiple instances of the API Gateway.
  * **API Gateway:** A single-entry point for all client requests, handling routing, authentication, and rate limiting.
  * **Microservices:**
      * **User Service:** User authentication, authorization, profile management.
      * **Product Catalog Service:** Product information, categories, search.
      * **Inventory Service:** Real-time stock levels, reservations.
      * **Cart Service:** Manages shopping carts, adding/removing items.
      * **Order Service:** Order creation, status updates, history.
      * **Payment Service:** Handles payment processing with third-party gateways.
      * **Shipping Service:** Integrates with shipping carriers, tracking.
      * **Notification Service:** Sends emails, SMS, push notifications.
      * **Review & Rating Service:** User reviews, product ratings.
  * **Caching Layer (Redis/Memcached):** For frequently accessed data (product details, user sessions, inventory snapshots).
  * **Message Queue/Event Bus (Kafka/RabbitMQ):** For asynchronous communication between microservices, ensuring loose coupling and resilience.
  * **Data Stores:**
      * **Relational Databases (PostgreSQL/MySQL):** For transactional data (Users, Orders, Inventory).
      * **NoSQL Databases (MongoDB/DynamoDB):** For flexible data (Product Catalog, User Profiles, Reviews).
      * **Search Engine (Elasticsearch):** For powerful product search capabilities.
  * **Monitoring & Logging (Prometheus/Grafana, ELK Stack):** For system health, performance, and troubleshooting.

### 3.2 UML DIAGRAM

[Image of: High-Level UML Diagram showing data flow from Clients to Microservices and Data Stores]

### 3.3 Service Breakdown

| Service Name | Primary Responsibility | Key APIs | Data Store |
| :--- | :--- | :--- | :--- |
| **Flash Sale Service** | Manages sale lifecycle, rules, and item assignment. | GET /active, POST /start, POST /end | Sql Server |
| **Inventory Service** | Manages real-time stock levels, reservations, and final locks. (Critical Path) | GET /stock, POST /reserve, POST /lock | Sql Server |
| **Order Service** | Handles order creation, status, and history. | POST /purchase, GET /user/{id}/orders | Sql Server |
| **Catalog Service** | Manages product information (SKUs, descriptions, pricing). | GET /products/{id}, GET /flash-sale-products | Sql Server |
| **Notification Service** | Sends real-time and asynchronous user notifications (Email, SMS, Push). | Internal API for triggering notifications. | Sql Server |
| **User Service** | Handles user authentication, authorization, and profile management. | POST /login, GET /profile, POST /register | Sql Server |
| **Payment Service** | Interfaces with third-party payment gateways for transaction processing. | POST /process, GET /status/{txnId} | Sql Server |

-----

## 4\. DATA DESIGN

### 4.1 FLASH SERVICE SCHEMA

```sql
CREATE TABLE flash_sales (
    sale_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    end_time TIMESTAMP WITH TIME ZONE NOT NULL,
    status VARCHAR(50) NOT NULL, -- e.g., 'SCHEDULED', 'ACTIVE', 'ENDED'
    max_purchases_per_user INT DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE flash_sale_items (
    sale_item_id UUID PRIMARY KEY,
    sale_id UUID REFERENCES flash_sales(sale_id),
    product_id UUID NOT NULL,
    sale_price DECIMAL(10, 2) NOT NULL,
    initial_quantity INT NOT NULL,
    CONSTRAINT unique_sale_product UNIQUE (sale_id, product_id)
);

-- Table to track participation limits for strong consistency check
CREATE TABLE flash_sale_participants (
    sale_id UUID REFERENCES flash_sales(sale_id),
    user_id UUID NOT NULL,
    orders_count INT DEFAULT 0,
    PRIMARY KEY (sale_id, user_id)
);
```

### 4.2 INVENTORY SERVICE SCHEMA

```sql
CREATE TABLE inventory (
    product_id UUID PRIMARY KEY,
    current_stock INT NOT NULL,
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Table for temporary reservations tied to an order attempt
CREATE TABLE inventory_reservations (
    reservation_id UUID PRIMARY KEY,
    order_id UUID NOT NULL, -- Link to pending order
    product_id UUID REFERENCES inventory(product_id),
    quantity INT NOT NULL,
    reserved_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL, -- TTL for cleanup
    status VARCHAR(50) NOT NULL -- 'PENDING', 'COMMITTED', 'CANCELLED'
);
```

### 4.3 CACHING STRATEGY

| Key Pattern | Value | TTL | Purpose |
| --- | --- | --- | --- |
| `flash_sale:{sale_id}` | `{"sale_info": "...", "remaining_stock": "...", "status": "..."}` | 300 seconds | Full details of an active sale for high-speed read. |
| `flash_sale_inventory:{sale_id}:{product_id}` | INTEGER (Remaining Flash Sale Stock) | 60 seconds | Real-time, atomic stock check. Decremented/Incremented atomically (INCRBY/DECRBY). |
| `user_session:{user_id}` | JWT/Session Data `{"token": "...", "roles": "..."}` | 3600 seconds | User session validation and authorization data. |

### 4.4 CDN CACHE STRATEGY

| Static Content Caching | Cache Duration |
| --- | --- |
| Product images: | Cache duration 1 week (max-age) |
| Sale banners: | Cache duration 1 hour (revalidate on change) |
| JavaScript/CSS: | Cache duration 1 year (versioned URLs) |
| **Dynamic Content** | **Cache Strategy** |
| Product availability: | Cache duration 1-5 seconds (Edge caching) |
| Sale countdown: | No cache (client-side render with server time sync) |

### 4.5 KEY ENTITIES

  * [Purchase Request]: (UserID, ProductID, Quantity)
  * [CurrentQuantity]: Atomic integer for stock (in Redis)
  * [Stock Reserved Confirmation]: Success from Redis
  * [Stock Sold Out Notification]: Failure from Redis
  * [Initial Order Record]: (OrderID, UserID, ProductID, Quantity, Status= 'PENDING\_RESERVATION')
  * [OrderCreated Event]: (OrderID, UserID, ProductID, Quantity, etc.)
  * [Payment Request]: (OrderID, Amount, PaymentDetails)
  * [Payment Result]: (Status, TransactionID)
  * [Order Status (Paid/Failed)]: Update to Order entity
  * [Order Confirmation]: Message back to Customer
  * [OrderConfirmed Event]: (OrderID, Status, PaymentTxnID, etc.)

### 4.6 DATA FLOW DIAGRAM

[Image of: Data Flow Diagram - Flash Sale Checkout Process]

-----

## 5\. API DESIGN

This API design focuses on the critical path of a flash sale, prioritizing high-read throughput for status and strict control for the purchase process.

### 5.1 RESTFUL API SPECIFICATION

#### 1\. Flash Sale Read & Status APIs (High Throughput)

These APIs are designed to be heavily cached (at CDN and Redis layers) to handle massive read traffic.

  * **GET /api/v1/sales/active**
      * **Purpose:** Retrieve a list of all currently active flash sale events.
      * **Caching:** High TTL (e.g., 30 seconds) at API Gateway/Redis.
  * **GET /api/v1/sales/{saleId}**
      * **Purpose:** Get full details of a specific sale, including its start/end times and key product information.
      * **Caching:** High TTL (e.g., 60 seconds).
  * **GET /api/v1/sales/{saleId}/status**
      * **Purpose:** Retrieve real-time remaining stock and current sale status (e.g., 'Active', 'Ending Soon', 'Sold Out').
      * **Caching:** Very low TTL (e.g., 1 second) or near-real-time updates via WebSockets for the countdown.

#### 2\. Inventory & Purchase APIs (Critical Path)

These APIs are highly sensitive, protected by rate limiting, and typically bypass deep caching. They trigger the distributed transaction (Saga).

  * **POST /api/v1/sales/{saleId}/purchase**
      * **Purpose:** Initiate the purchase process by attempting to reserve stock and starting the order flow.
      * **Request Body:** `{"productId": "UUID", "quantity": 1}`
      * **Response:** 202 Accepted + `{"orderId": "UUID", "status": "RESERVATION_PENDING"}`. The actual payment/fulfillment is handled asynchronously.
      * **Security:** Aggressive Rate Limiting (e.g., 1 request per user per minute).

#### 3\. Order and Fulfillment APIs

These APIs track the status of the asynchronous purchase flow.

  * **GET /api/v1/orders/{orderId}/status**
      * **Purpose:** Check the current status of an order that was initiated via the purchase API.
      * **Response:** `{"orderId": "UUID", "status": "PAYMENT_PENDING"}` or `{"status": "COMPLETED"}`.
      * **Security:** Requires User Authentication and ownership verification.
  * **POST /api/v1/payments/process (Internal/Webhook)**
      * **Purpose:** API for the Payment Service or external gateway to notify the system of a payment outcome.
      * **Security:** Highly secured, often protected by IP whitelisting or cryptographic signatures.

#### 4\. Management APIs (Admin Only)

These APIs are used by internal staff, secured by strong role-based access control (RBAC), and are not performance-critical.

  * **POST /api/v1/admin/sales**
      * **Purpose:** Create a new flash sale event (sets rules, links product, defines quantity).
      * **Security:** Admin role required. Triggers Cache Warming upon success.
  * **POST /api/v1/admin/sales/{saleId}/start**
      * **Purpose:** Manually start a scheduled sale (optional: usually automated by a scheduler).
  * **GET /api/v1/admin/inventory/reservations**
      * **Purpose:** Review all active stock reservations and their expiry times.
      * **Security:** Admin role required.

### 5.2 UML Component Diagram

[Image of: UML Component Diagram: .NET Core E-commerce Backend]

### 5.3 API Gateway and Request Flow

The API Gateway acts as the crucial entry point, managing the initial load and security before requests hit the core services.

  * **Load Balancer (L4/L7):** Manages overall traffic and routes it to the API Gateway instances.
  * **API Gateway:**
      * **Authentication & Authorization:** Validates JWT/OAuth tokens from clients via the User Service.
      * **Rate Limiting:** Protects services from abuse and DDoS attacks.
      * **Routing:** Forwards requests to the correct microservice cluster based on the path (e.g., /products/\* $\rightarrow$ Catalog Service).
      * **Edge Caching:** Caches common responses (e.g., general product lists).

**Example Flow (Synchronous):**

1.  Client sends `GET /products/123`.
2.  API Gateway authenticates and routes to Product Catalog Service.
3.  Catalog Service checks Redis Cache for `product:123`.
4.  If cache miss, fetches from NoSQL DB.
5.  Returns product data to the client via the Gateway.

### 5.4 Folder Structure

```md
order-service/
├── config/
│   ├── default.json
│   └── production.json
├── src/
│   ├── api/
│   │   ├── controllers/
│   │   │   └── order.controller.js
│   │   ├── routes/
│   │   │   └── order.routes.js
│   │   └── index.js
│   ├── domain/
│   │   ├── events/
│   │   │   └── order.events.js
│   │   ├── models/
│   │   │   └── order.model.js
│   │   └── services/
│   │       └── order.service.js
│   ├── infrastructure/
│   │   ├── external/
│   │   │   ├── inventory.client.js
│   │   │   └── payment.client.js
│   │   ├── messaging/
│   │   │   └── event.consumer.js
│   │   └── repository/
│   │       └── order.repository.js
│   └── app.js
├── tests/
│   ├── e2e/
│   ├── integration/
│   └── unit/
├── Dockerfile
└── package.json
```

### 5.5 Request Processing Flow

[Image of: Request Processing Flow sequence diagram]

### 5.6 EVENT DRIVEN ARCHITECTURE

| Domain Events | Payload | Publishers | Subscribers |
| --- | --- | --- | --- |
| FlashSaleStarted | `{"saleId": "...", "productId": "...", "startTime": "...", "duration": "..."}` | Flash Sale Service | Notification, Catalog, Inventory |
| InventoryReserved | `{"orderId": "...", "userId": "...", "productId": "...", "quantity": 1}` | Inventory Service | Order, Payment |
| PurchaseCompleted | `{"orderId": "...", "userId": "...", "amount": "...", "paymentTxnId": "..."}` | Payment Service | Order, Inventory, Notification, Analytics |
| ReservationFailed | `{"orderId": "...", "reason": "...", "userId": "..."}` | Inventory Service | Order, Notification |

-----

## 6\. Scalability & Performance

### 6.1: Load Distribution Strategy

**1. API Gateway (YARP) Configuration:**

  * **Routing strategy:** Path-based routing (e.g., /api/flash-sales to Flash Sale Service cluster).
  * **Load balancing algorithm:** Least Request (Distributes load based on current active connections).
  * **Health check configuration:** Active health checks on /health endpoint for all microservices.
  * **Circuit breaker settings:** Implement to isolate failing services (e.g., Payment Service) and prevent cascading failures.

**2. Database Load Distribution:**

  * **Read replicas strategy:** Use multiple Read Replicas (5-10+) for high-volume read traffic (Catalog, Order Status).
  * All read traffic bypasses the primary DB.
  * **Write scaling approach:** Primary database handles all critical writes (Orders, Reservations). Optimize with connection pooling and batched writes where possible.
  * **Sharding strategy (if applicable):** Shard high-volume tables (e.g., orders, inventory\_reservations) based on user\_id or sale\_id to distribute I/O load.

### 6.2: Performance Optimization

| Component | Potential Bottleneck | Optimization Strategy | Expected Improvement |
| --- | --- | --- | --- |
| Database Writes | `INSERT into orders` and `inventory_reservations` tables. | Use a Message Queue (Kafka) for async write/commit, only doing atomic Redis updates on the critical path. | 10x higher TPS |
| Inventory Checks | Concurrent checks/updates on remaining stock. | Atomic operations (INCR/DECR) on Redis for real-time flash sale inventory. | \<1ms latency |
| User Authentication | Repeated token validation and profile lookups. | JWT for stateless authentication; Cache user sessions/roles in Redis. | 50% latency reduction |
| Payment Processing | Latency of external payment gateway API calls. | Move payment processing off the critical path to be asynchronous via event-driven (Saga). Use Circuit Breaker. | Improved purchase completion time |

-----

## 7\. Consistency & Reliability

### 7.1 ACID vs BASE Trade-offs:

**Strong Consistency Requirements (ACID):**

  * **Inventory management:** Why? Prevent overselling (the single most critical failure point).
  * Requires atomic operations or distributed locks/transaction (Saga) to guarantee stock level integrity.
  * **Payment processing:** Why? Financial integrity. Payment must be reliably recorded and tied to the order.
  * Requires transactional guarantees with the payment gateway.

**Eventual Consistency Acceptable (BASE):**

  * **User notifications:** Why? A slight delay (seconds) in sending a confirmation email or push notification is acceptable and doesn't affect the core business transaction.
  * **Analytics/reporting:** Why? Data is ingested asynchronously into a warehouse. Reports can tolerate data being slightly behind the current state for improved ingestion performance.

### 7.2 Failure Handling

**Circuit Breaker Configuration:**

| Service | Failure Threshold | Timeout | Fallback Strategy |
| --- | --- | --- | --- |
| Payment Service | 5 consecutive failures | 500ms | Return a `503 Service Unavailable` error and allow the Saga to compensate (un-reserve inventory). |
| Inventory Service | 10% failure rate over 60 seconds | 100ms | If for non-flash-sale stock, return a safe "out of stock" response. If flash-sale, rely on Redis atomic checks first. |

-----

## 8\. Security & Compliance

### 8.1 Authentication & Authorization:

**User Authentication:**

  * **Method:** OAuth 2.0 with JWT (JSON Web Tokens).
  * **Token expiry:** 1 hour (Access Token), 1 week (Refresh Token).
  * **Rate limiting per user:** 5 login attempts per 30 minutes to prevent brute-force attacks.

**API Security:**

  * **Authentication scheme:** JWT validation and authorization check for every request at the API Gateway.
  * **Rate limiting strategy:** Global rate limiting on API Gateway (e.g., 200 RPS per IP) and specific per-user/per-API limits (e.g., purchase API).
  * **DDoS protection:** Use a cloud-based Web Application Firewall (WAF) like AWS WAF or Cloudflare.

### 8.2 Data Protection

**PII Data Handling:**

  * **Encryption at rest:** AES-256 using disk/database encryption keys (KMS/Secrets Manager).
  * **Encryption in transit:** TLS 1.3 across all services and external communication.
  * **Data retention policy:** Purge PII after 7 years or per regulatory requirements.

**Payment Data:**

  * **PCI DSS compliance approach:** Outsource payment storage and processing to a PCI DSS Level 1 compliant provider (e.g., Stripe, Adyen).
  * **Tokenization strategy:** Only store a non-sensitive payment token (provided by the payment gateway) in the Order Service database, never raw card details.

### 8.3 Fraud Prevention

**Bot Detection (Crucial for Flash Sales):**

  * **Detection Methods:**
      * Rate limiting: Aggressive limits on purchase and status check APIs.
      * CAPTCHA integration: Implement reCAPTCHA v3 on the purchase endpoint, only blocking suspicious users/traffic.
      * Behavioral analysis: Analyze mouse/touch movement, page load times, and request patterns (e.g., perfectly timed requests, uniform intervals).
  * **Prevention Measures:**
      * Account verification: Email and SMS verification required before flash sale participation.
      * Purchase limits: Strict one purchase per user/IP/address enforced by the Flash Sale Service/Inventory Service.
      * Suspicious activity handling: Flag users/IPs for manual review, shadow-ban bot traffic, and implement immediate temporary IP bans.

-----

## 9\. MONITORING AND OBSERVABILITY

### 9.1 METRICS AND MONITORING

| Metric | Purpose | Target Value | Alert Threshold |
| --- | --- | --- | --- |
| Sale conversion rate | Measure successful purchases vs. page views. | \>5% | \<1% (Immediate investigation) |
| Average response time | Gauge system performance under load. | \<100ms (P95) | \>300ms (Auto-scale trigger) |
| Inventory accuracy | Check alignment between Redis and DB stock levels. | 100% | \<100% (Critical PagerDuty alert) |
| Payment success rate | Monitor external service health and internal errors. | \>99% | \<95% (Payment Service failure alert) |

**Technical Metrics:**

  * **Application Performance:**
      * Request latency P99: \<500ms
      * Throughput (RPS): Expected Peak RPS + 20%
      * Error rate: \<0.1%
  * **Infrastructure Metrics:**
      * CPU utilization: \<70% (Auto-scale trigger)
      * Memory usage: \<80%
      * Database connections: \<80% of max pool size

### 9.2 LOGGING AND TRACING

**Logging Strategy:**

  * **Log Levels and Content:**
      * **ERROR:** Unhandled exceptions, failed external calls (e.g., Payment Gateway error). Includes stack trace.
      * **WARN:** Recoverable issues, slow database queries, Circuit Breaker open events.
      * **INFO:** Key business operations (e.g., Order Created, Inventory Reserved, User Login).
      * **DEBUG:** Detailed internal flow logic, request/response headers.
  * **Structured Logging Format:** JSON structure for easy ingestion and querying (ELK/Loki stack).

**Distributed Tracing:**

  * **Trace Correlation:**
      * **Trace ID generation:** Generated by the API Gateway on the first request.
      * **Cross-service propagation:** Propagated using W3C Trace Context headers by every service in the chain.
  * **Sampling strategy:** Head-based sampling (e.g., sample 100% of requests during a flash sale, and 5% during normal load).

-----

## 10\. DevOps Structure

### 10.1 Key Pillars of DevOps for Flash Sales:

  * **Continuous Integration (CI):** Automate builds, unit tests, and code quality checks with every code commit.
  * **Continuous Delivery/Deployment (CD):** Automate the release process, pushing validated changes through various environments up to production.
  * **Infrastructure as Code (IaC):** Manage all infrastructure (servers, databases, networks) through code, enabling consistency and version control.
  * **Containerization & Orchestration:** Package applications in isolated containers (Docker) and manage them with an orchestrator (Kubernetes).
  * **Monitoring & Logging:** Centralized tools to observe system health, performance, and application behavior in real-time.
  * **Security Integration:** Embed security practices throughout the entire pipeline (DevSecOps).
  * **Automated Testing:** Comprehensive testing at multiple levels (unit, integration, performance, security).
  * **Disaster Recovery (DR) & Backup:** Strategies for data protection and quick recovery from failures.

### 10.2 High-Level DevOps Workflow

```graph
graph TD
    A [Developers] --> B (Code Commit);
    B --> C {Version Control<br>(Git, GitHub/GitLab)};
    C --> D [CI Server<br>(Jenkins, GitLab CI, GitHub Actions)];
    D --> E {Build & Test};
    E -- Success --> F [Container Registry<br>(Docker Hub, ECR)];
    F --> G [CD Server<br>(ArgoCD, Spinnaker)];
    G --> H {Infrastructure as Code<br>(Terraform, Helm)};
    H --> I [Cloud Provider<br>(AWS, Azure, GCP)];
    I --> J [Kubernetes Cluster];
    J --> K [Microservices<br>Running in Pods];
    K --> L [Monitoring & Alerting<br>(Prometheus, Grafana)];
    K --> M [Logging & Tracing<br>(ELK, Jaeger)];
    L -- Alerts --> Ops (Operations Team);
    M -- Insights --> Dev (Development Team);
    Ops <--> Dev;
```


### 10.3 UML Deployment & Component

[Image of: UML Deployment & Component diagram for Production Environment]

### 10.4 Infrastructure as Code (IaC)

**Container Strategy:**

  * **Containerization Approach:** Docker for all microservices.
    * Base images: Minimal, security-hardened images (e.g., Alpine or Distroless).
    * Multi-stage builds: Used to minimize final image size by discarding build dependencies.
    * Resource limits: Hard limits for CPU and Memory defined in the orchestration platform to prevent noisy neighbor issues.

  * **Orchestration: Kubernetes (EKS/GKE/AKS)**

    * Scaling strategy: Horizontal Pod Autoscaler (HPA) based on CPU utilization and Requests Per Second (RPS) custom metrics. KEDA for scaling based on Kafka queue depth.
    * Rolling update strategy: Max unavailable set to 25%, minimum available set to 75% for a zero-downtime deployment.

**Environment Strategy:**

  * **Development Environment:**
      * Local development setup: Docker Compose to run local dependencies (DB, Redis, Kafka) and the service code.
      * Database seeding: Scripts run automatically on container start for initial dev data.
      * Service mocking: WireMock or similar for mocking external services (e.g., Payment, Notification).
 
 * **Production Environment:**
      * Auto-scaling triggers: HPA on CPU \> 70%, RPS \> 1000. Scale up and maintain 20% buffer capacity.
      * Resource allocation: Dedicated namespaces/clusters for isolation

### 10.5 CI/CD Pipeline

| Stage | Trigger | Actions |
| --- | --- | --- |
| **1. Code Commit** | Git push to `main` branch. | Linter/Static Analysis, Security Scanning (SAST). |
| **2. Build & Test** | **Unit tests:** Run all tests (target \>85% coverage).<br>**Integration tests:** Service-to-service communication tests on a shared testing DB.<br>**Performance tests:** Run a smoke load test (low-volume). | |
| **3. Staging Deployment** | Manual approval after Build & Test success. | **Validation steps:** Deploy to Staging environment.<br>**Smoke tests:** Basic API calls to ensure services are functional. |
| **4. Production Deployment** | Manual/Scheduled approval from Staging. | **Deployment strategy:** Canary Deployment (10% traffic to new version for 1 hour).<br>**Rollback plan:** Automatic rollback if error rate spikes (\>1%) or latency increases (\>10%).<br>**Health checks:** Final check on `/health` and business-critical endpoints. |

-----

## 11\. Capacity Planning

### 11.1: Load Estimation

  * Expected concurrent users
  * Requests per second (RPS)
  * Database transactions per second
  * Cache operations per second

### 11.2: Cost Optimization

  * **Reserved instances:** Purchase 1-3 year reserved instances for the baseline, always-on load (e.g., Catalog, User, DB primary).
  * **Auto-scaling policies:** Aggressively scale-in immediately after the sale peak to save on compute costs.
  * **Resource right-sizing:** Continuously monitor resource utilization (CPU/Memory) in off-peak times to adjust Kubernetes resource requests/limits and prevent over-provisioning.

-----

## 12\. TESTING

### 12.1 Test Plan Design

**Unit Tests:**

  * **Coverage target:** \>85%
  * **Test categories:** Business logic, data transformations, utility functions.
  * **Mock strategies:** Mock dependencies (Repository, other Service Clients) to isolate the service under test.

**Integration Tests:**

  * **Service-to-service:** Test synchronous API calls between services (e.g., Flash Sale calling Inventory).
  * **Database integration:** Test actual repository calls against a test container database (e.g., Testcontainers).
  * **External API integration:** Test against mocked external services (Payment Gateway) using WireMock.

**End-to-End Tests:**

  * **Critical user journeys:** Full path from Sale View -\> Purchase Button -\> Order Confirmed.
  * **Cross-browser testing:** Verify UI/UX on major browsers.
  * **Mobile responsiveness:** Test on various mobile devices/emulators.

### 12.2 Performance Testing

**1. Load Testing: The Steady State**
This test simulates the typical high traffic seen after the initial sale rush has stabilized. The primary goal is to verify the system's stability and endurance. Testers will gradually introduce virtual users performing common actions (browsing, adding to cart, checking sale status) and maintain this high load for a long duration. The test is successful if all system metrics, especially latency, remain consistently low and predictable, and the Error Rate is negligible for the entire period.

**2: Peak Testing (Viral Peak Rush)**
This is the most critical test, simulating the instantaneous, massive spike in purchase attempts when the sale starts. The goal is to confirm that the critical purchase path—specifically the inventory reservation—can handle maximum concurrency without failure. Testers will introduce a vast number of users very quickly (within seconds) who all try to execute the purchase transaction. The key success criterion is that the system must not crash or become unresponsive, and, most importantly, the system must guarantee zero overselling despite the extreme simultaneous demand.

**3: Stress Testing (Overload and Recovery)**
This test pushes the system beyond its expected capacity to discover its true limits. Testers will continuously increase the user load until performance metrics, such as transaction rate or latency, show a significant, irreversible drop—this is the breaking point. The secondary, but equally important, goal is to assess Recovery: once the load is removed, the system's automatic mechanisms (like auto-scaling and circuit breakers) must allow all services to scale down and stabilize gracefully without manual intervention.