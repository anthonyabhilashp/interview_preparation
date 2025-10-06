# 🧠 High-Level Design (HLD): Webhook Management System

---

## 1. Overview

The **Webhook Management System** provides reliable ingestion, storage, delivery, and observability of webhook events for customers.  
It allows clients (e.g., Shopify, Stripe, or internal apps) to register endpoints and receive event notifications asynchronously, with strong guarantees for **durability**, **retries**, and **observability**.

**Goal:** Deliver webhooks *reliably, securely, and at scale*, even under burst loads, while providing visibility into all events through APIs and dashboards.

---

## 2. Requirements

### ✅ Functional Requirements

- **User & Endpoint Management**
  - Users can register/login, create webhook endpoints, and manage API keys.
  - Endpoints can be activated, paused, or deleted.

- **Webhook Ingestion**
  - Accept inbound webhooks (HTTP POST) from external producers.
  - Validate authenticity (HMAC signature).
  - Store payloads and metadata.

- **Event Delivery**
  - Forward stored webhooks to configured customer destinations.
  - Ensure at-least-once delivery with retries and backoff.
  - Record delivery status and response details.

- **Replay & Monitoring**
  - Support manual or automated replay of failed deliveries.
  - Expose APIs and dashboards for delivery history, analytics, and endpoint health.

- **Notifications**
  - Generate alerts for repeated failures, endpoint downtime, or high error rates.

---

### ⚙️ Non-Functional Requirements

| Concern | Target / Approach |
|----------|------------------|
| **Scalability** | Handle millions of webhooks per day via horizontal scaling & partitioned queues |
| **Durability** | Persist before ack → at-least-once semantics |
| **Availability** | 99.9% uptime via stateless API and replicated storage |
| **Security** | HTTPS, HMAC verification, encryption at rest, API key authentication |
| **Latency** | Ingestion <100 ms, delivery <2 s average |
| **Observability** | End-to-end tracing, metrics, dashboards, alerts |
| **Cost Efficiency** | Object store for large payloads, compute autoscaling |

---

## 3. Architecture Overview

### Core Components

| Component | Responsibility |
|------------|----------------|
| **API Gateway / Ingress Service** | Receives incoming webhooks, performs HMAC validation, persists payload, and enqueues message |
| **Message Queue (Kafka)** | Decouples ingestion from delivery, provides durability and replay |
| **Delivery Worker Pool** | Fetches queued events, performs delivery with retries and backoff |
| **Metadata Store (PostgreSQL)** | Stores event metadata, endpoints, delivery attempts, users |
| **Payload Store (MinIO/S3)** | Stores raw webhook bodies for durability and auditing |
| **Dashboard / Management API** | Exposes analytics, replay API, and team management features |
| **Notification Service** | Generates alerts (email/Slack) for failures and health issues |
| **Monitoring Stack (Prometheus/Grafana)** | Collects and visualizes metrics, logs, alerts |
| **Auth Service (Keycloak)** | Manages authentication, RBAC, and API key issuance |

---

## 📊 High-Level Architecture Diagram

java
Copy code
            ┌─────────────────────────────┐
            │   External Systems (Producers)│
            │    e.g., Stripe, Shopify     │
            └───────────────┬──────────────┘
                            │  (HTTP POST)
                            ▼
             ┌────────────────────────────────┐
             │  API Gateway / Ingress Service  │
             │ (Go + Fast HTTP + HMAC verify)  │
             └──────────────┬─────────────────┘
                            │
            ┌───────────────┴────────────────────────────┐
            │ Persist Payload (MinIO)                    │
            │ Insert Metadata (Postgres)                 │
            │ Publish Message (Kafka Topic: “ingestion”) │
            └────────────────┬───────────────────────────┘
                             │
                             ▼
                ┌───────────────────────────┐
                │ Delivery Worker Pool (Go) │
                │  - Read from Kafka         │
                │  - Fetch payload (MinIO)   │
                │  - Deliver via HTTP POST   │
                │  - Retry on failure        │
                └──────────────┬────────────┘
                               │
                               ▼
         ┌────────────────────────────┐
         │  Target Endpoints (Clients)│
         │  e.g., https://api.client  │
         └────────────────────────────┘

                  ┌───────────────────────┐
                  │ Dashboard (React/Next)│
                  │ Monitoring + Replay   │
                  └───────────────────────┘
markdown
Copy code

---

## 4. Data Flow

### 1️⃣ Ingestion
1. Client (producer) POSTs event to `https://hooks.example.com/h/{endpoint_token}`  
2. Gateway verifies HMAC signature and timestamp  
3. Payload uploaded to MinIO (`/payloads/{event_id}.json`)  
4. Metadata written to PostgreSQL  
   - event_id, endpoint_id, received_at, s3_key, status=PENDING  
5. Message published to Kafka topic `webhooks.ingest`  
6. Return `202 Accepted` to producer  

### 2️⃣ Delivery
1. Worker reads from Kafka (`webhooks.ingest`)  
2. Fetches payload from MinIO  
3. Reads target URL and headers from endpoint configuration  
4. Sends POST request with payload (signed using endpoint’s secret)  
5. Updates delivery status in Postgres  
   - SUCCESS → mark completed  
   - FAILURE → schedule retry via delayed queue or reinsert to Kafka  
6. After 3–5 retries → mark as FAILED and notify via Notification Service  

### 3️⃣ Replay
- Dashboard user selects failed events and triggers replay.  
- Backend requeues those events manually with the same payload and event_id.  

---

## 5. Data Model (Simplified)

**Table: endpoints**
id (uuid)
user_id
name
target_url
secret_key
status (active/inactive)
created_at, updated_at

css
Copy code

**Table: events**
id (uuid)
endpoint_id
s3_key
status (PENDING | SUCCESS | FAILED)
received_at, updated_at

css
Copy code

**Table: deliveries**
id (uuid)
event_id
attempt_number
response_code
response_body_snippet
next_retry_at
error_message
created_at, updated_at

pgsql
Copy code

---

## 6. Scalability and Reliability

| Concern | Strategy |
|----------|-----------|
| **Horizontal Scaling** | Stateless services (Ingress + Workers) behind load balancer |
| **Hot Endpoint Isolation** | Partition Kafka by `endpoint_id`, ensuring independent backpressure handling |
| **Retries** | Exponential backoff with jitter (e.g., 1m → 5m → 15m) |
| **At-Least-Once Delivery** | Persist before ACK + idempotent event IDs |
| **High Availability** | Multi-instance deployment, DB replication, MinIO cluster |
| **Disaster Recovery** | Periodic DB + object backups, cross-region replication |
| **Monitoring** | Track queue lag, error rates, throughput, latency via Prometheus |
| **Dead Letter Queue** | Route permanently failed events for later replay |

---

## 7. Security

- All communication via **HTTPS/TLS 1.3**  
- **HMAC verification** for inbound events using per-endpoint secrets  
- **Signed outgoing deliveries** (`X-Hub-Signature`)  
- **API key & JWT-based authentication** for management APIs  
- **RBAC** for teams and organizations via Keycloak  
- **Data encryption at rest** (Postgres + MinIO KMS)  
- **Rate limiting & throttling** per tenant to prevent abuse  

---

## 8. Observability

- **Metrics:** Ingestion rate, delivery success %, queue lag, retry count, avg latency  
- **Logs:** Structured JSON logs with correlation IDs (event_id)  
- **Tracing:** OpenTelemetry spans (Ingress → Kafka → Delivery → Response)  
- **Dashboards:** Grafana panels for success rate, latency, failure by endpoint  
- **Alerts:**  
  - Failure rate > 5% → Alert  
  - Kafka lag > 10k → Alert  
  - Delivery latency > 2s (p95) → Alert  

---

## 9. Technology Stack

| Layer | Technology | Justification |
|--------|-------------|---------------|
| **Language / Runtime** | Go | High performance, lightweight concurrency |
| **Queue** | Kafka | Partitioned, durable, scalable messaging |
| **Database** | PostgreSQL | ACID transactions, relational queries |
| **Object Storage** | MinIO / S3 | Durable, cost-effective blob storage |
| **Gateway / LB** | Kong / Nginx | TLS termination, routing, rate limiting |
| **Auth** | Keycloak | Open-source IAM solution |
| **UI** | React / Next.js | Dynamic dashboard, API integrations |
| **Monitoring** | Prometheus + Grafana | Metrics + alerting |
| **Deployment** | Kubernetes | Autoscaling, rolling updates |
| **CI/CD** | GitHub Actions | Automated testing and deployment |

---

## 10. Trade-offs and Considerations

| Decision | Trade-off |
|-----------|------------|
| **Kafka vs RabbitMQ** | Kafka offers better partitioning & replay; RabbitMQ simpler for small-scale ops |
| **At-least-once vs Exactly-once** | Chose at-least-once for simplicity; idempotency ensures safety |
| **Postgres + MinIO Split** | Adds coordination complexity but improves cost/performance |
| **Single-region vs Multi-region** | Start single-region; add cross-region replication for enterprise SLA |

---

## 11. Future Enhancements

- Multi-region delivery optimization (region-local workers)  
- Dynamic retry strategies based on failure type  
- Rate limiting per endpoint to protect misbehaving customers  
- Payload transformation hooks (filtering/mapping)  
- Webhook signing certificate rotation for enhanced security  
- Machine learning–based failure pattern detection for proactive alerting  

---

## 12. Summary

> The **Webhook Management System** is a scalable, reliable, and secure platform designed to handle millions of webhook events daily.  
> It guarantees at-least-once delivery, provides rich observability, supports replay and alerting, and scales horizontally with Kafka, Go-based workers, and distributed storage.  
> The architecture prioritizes durability, modularity, and cost-efficiency — making it suitable for production-grade SaaS or enterprise systems.

---

## 🧩 Mermaid Diagrams for Webhook Management System

> 📘 **Note:** GitHub supports Mermaid rendering natively.  
> Copy-paste these blocks directly into your Markdown file to visualize the diagrams.

---

### 🏗️ High-Level Architecture

```mermaid
graph TD

subgraph External["External Systems (Producers)"]
  A1["Stripe / Shopify / Payment Gateways"]
end

subgraph Ingress["API Gateway / Ingress Service (Go)"]
  B1["HTTP Endpoint /h/{token}"]
  B2["HMAC Verification"]
  B3["Persist Payload (MinIO)"]
  B4["Insert Metadata (PostgreSQL)"]
  B5["Publish to Kafka (ingestion topic)"]
end

subgraph Queue["Message Queue"]
  C1["Kafka Topic: webhooks.ingest"]
end

subgraph Delivery["Delivery Worker Pool (Go)"]
  D1["Consume from Kafka"]
  D2["Fetch Payload (MinIO)"]
  D3["POST to Target Endpoint"]
  D4["Update Status (PostgreSQL)"]
  D5["Retry on Failure / Backoff"]
end

subgraph Storage["Storage Layer"]
  E1["PostgreSQL - Metadata & Deliveries"]
  E2["MinIO - Raw Payloads"]
end

subgraph Client["Target Systems (Consumers)"]
  F1["Customer Endpoint URL"]
end

subgraph Dashboard["Dashboard / Monitoring"]
  G1["React / Next.js UI"]
  G2["Prometheus + Grafana"]
  G3["Notification Service (Email/Slack)"]
end

A1 -->|Webhook POST| B1
B1 --> B2 --> B3 --> B4 --> B5 --> C1
C1 --> D1 --> D2 --> D3 --> F1
D3 -->|Response| D4
D4 --> E1
D1 --> E2
D4 --> G1
D5 --> C1
G2 --> G3
🔄 Data Flow: Event Lifecycle
mermaid
Copy code
sequenceDiagram
    participant Producer as External System (e.g., Stripe)
    participant Gateway as Ingress Service
    participant Kafka as Kafka Topic
    participant Worker as Delivery Worker
    participant DB as PostgreSQL
    participant Storage as MinIO
    participant Client as Target Endpoint

    Producer->>Gateway: HTTP POST /h/{token} (payload + HMAC)
    Gateway->>Gateway: Verify HMAC Signature
    Gateway->>Storage: Upload Payload
    Gateway->>DB: Insert Metadata (status=PENDING)
    Gateway->>Kafka: Publish event_id
    Gateway-->>Producer: 202 Accepted

    Worker->>Kafka: Consume event_id
    Worker->>Storage: Fetch Payload
    Worker->>Client: POST Payload to target_url
    Client-->>Worker: 200 OK / 5xx / Timeout
    Worker->>DB: Update status SUCCESS / FAILED
    Worker->>Kafka: Requeue for Retry (if failed)
