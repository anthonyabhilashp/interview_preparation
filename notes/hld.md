
# 🧠 High-Level Design (HLD) Cheat Sheet

A High-Level Design (HLD) explains how a system works as a whole — focusing on architecture, components, data flow, and trade-offs, not code-level details.

---

## 🧱 What Does an HLD Cover?

An effective HLD should explain:
- **Major system components**
- **How components communicate**
- **Key data flows between components**
- **Technologies or patterns used**
- **How the design meets scalability, reliability, and performance goals**

---

## ✅ Functional vs ⚙️ Non-Functional Requirements

### ✅ Functional Requirements (FR)
Functional requirements describe what the system should do — its features and expected behaviors.

**Examples:**
- Users can register, update, and delete accounts
- System processes and validates user requests
- Users can access or modify their data
- Admins can manage users and generate reports
- The system enforces permissions and business rules

> Think of FRs as “features and workflows.”

### ⚙️ Non-Functional Requirements (NFR)
Non-functional requirements describe how well the system performs — the quality attributes of the system.

**Examples:**
- **Scalability:** Should handle thousands of concurrent requests/users
- **Reliability:** Must recover gracefully from failures (with retries and fault tolerance)
- **Latency:** Average response time < 2 seconds
- **Durability:** No data loss even after restarts or crashes
- **Security:** All requests must be authenticated and authorized
- **Observability:** Centralized logging, metrics, and alerts
- **Availability:** Maintain 99.9% uptime

> Think of NFRs as “performance, reliability, and quality guarantees.”

---

## 🧭 7-Step Framework for HLD Interviews

You can use this framework for any system design question — e.g., Netflix, Instagram, Uber, or any SaaS product.

### 1. Understand the Problem

Before designing, clarify:
- Who are the users?
- What are the main use cases?
- What are the scale or performance constraints?
- What are the SLAs (latency, uptime, etc.)?

> Example: “Do we expect millions of users or just a few thousand?”

### 2. Define Functional & Non-Functional Requirements

List both clearly to demonstrate structured thinking and scope understanding.

### 3. Estimate Scale (Capacity Planning)

Make rough calculations for:
- Requests per second / concurrent users
- Data size and growth rate
- Storage or throughput requirements

Helps justify technology choices (e.g., DB type, caching, message queues).

### 4. Identify Core Components (Architectural Breakdown)

Decompose the system into logical building blocks. Typical categories:
- API Gateway / Load Balancer
- Application Service Layer / Business Logic
- Database / Storage Layer
- Cache / CDN
- Asynchronous Processing (e.g., Queues, Workers)
- Monitoring / Logging / Analytics

**Example (Generic Architecture):**
```
[Client / UI] → [API Gateway] → [Application Service] → [Database]
                                   ↓
                               [Cache / Queue / Analytics]
```

### 5. Explain the Data Flow

Describe the typical sequence of operations:
- User sends a request via API/UI
- Request hits the gateway → validated → forwarded to service
- Business logic executes (reads/writes data)
- Data stored or retrieved from DB/cache
- Response returned to client
- Logs and metrics recorded for monitoring

### 6. Choose Appropriate Technologies

Base your choices on trade-offs and requirements.

**Examples:**
- Frontend: React / Next.js / Flutter
- Backend: Node.js / Go / Java / Python
- Database: PostgreSQL / MongoDB / Cassandra
- Cache: Redis / Memcached
- Queue: Kafka / RabbitMQ / SQS
- Monitoring: Prometheus / Grafana / ELK

Always explain why you chose a specific technology (e.g., “Redis for low-latency caching,” “Kafka for high-throughput event streaming”).

### 7. Address NFRs and Design Trade-offs

Demonstrate engineering judgment:

| Concern       | Design Approach                        |
|-------------- |----------------------------------------|
| Scalability   | Horizontal scaling, load balancing      |
| Reliability   | Replication, retry mechanisms           |
| Durability    | Persistent storage, backup strategies   |
| Performance   | Caching, indexing, CDN                  |
| Security      | Auth, encryption, least privilege       |
| Observability | Centralized logs, metrics, tracing      |
| Trade-offs    | Balance latency vs cost, simplicity vs flexibility |

💬 Bonus: The Interview Mindset
When explaining your HLD, follow this structure:
“Let’s start by defining the requirements.
Then I’ll describe the core architecture and data flow,
and finally, I’ll discuss scalability, reliability, and trade-offs.”

That structure instantly signals clarity and strong design thinking.

📘 Pro Tip: Practice Systems
To master HLD, try designing:
URL Shortener
Chat App (like WhatsApp)
File Storage (like Google Drive)
Social Feed System (like Twitter)
Notification / Email Service
E-commerce Checkout Flow
Apply the same 7-step approach to each one.