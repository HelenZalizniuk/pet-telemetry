# Data Access Strategy & Communication Patterns

This document specifies how data flows between system components, the communication protocols used, and the transition from standard messaging to high-throughput streaming.

## 1. Data Communication Scenarios

| Scenario | Connection Type | Protocol | Delivery Mode | Rationale |
| :--- | :--- | :--- | :--- | :--- |
| **Telemetry Ingestion** (Collar -> Go) | Streaming / Messaging | **MQTT** | **Ephemeral** | High throughput (2,000+ RPS). One lost packet is non-critical as the next arrives in 5s. |
| **Main Data Stream** (Go -> Storage/Proc) | Streaming | **Apache Kafka** | **Durable / Replayable** | Acts as a high-speed buffer and "source of truth" for all telemetry before it hits the DB. |
| **Critical Alerts** (Go -> Notification) | Messaging | **NATS / Kafka** | **Durable** | Health alerts (arrhythmia, etc.) must never be lost. Durable queues ensure delivery. |
| **Profile & Registry** (BFF -> Java) | RPC | **gRPC** | **Unary Call** | Low latency for transactional operations (registration, medical norms) using Protobuf. |
| **Vitals Threshold Sync** (Java <-> Go) | Shared Access | **Redis** | **Side-cache** | Zero-latency access to pet-specific health limits to avoid DB bottlenecks. |

## 2. Architectural Evolution: Why Kafka?

While initial designs (Task-2/3) considered **RabbitMQ** for simple notifications, the requirement for **2,000+ RPS** and **172M messages/day** necessitated a shift to **Apache Kafka**.

* **Throughput:** Kafka handles massive sequential writes to disk, allowing the Go Ingestor to offload data instantly.
* **Backpressure Handling:** Kafka serves as a "shock absorber." If TimescaleDB experiences high latency during maintenance, Kafka buffers the incoming stream without losing data.
* **Data Replayability:** Unlike RabbitMQ, Kafka retains data. If we deploy a new AI-analysis service, it can "replay" the last 24 hours of vitals to train its model.

## 3. Resilience & Failover Mechanisms

To maintain 99.9% availability, we implement the following safety layers:

### A. Shared Cache (Redis) Failure
* **Mechanism:** Go service retains an **In-memory LRU Cache** of the last 1,000 active thresholds.
* **Fallback:** Use breed-standard "safe zones" (e.g., 60-140 bpm) if no specific data is found.

### B. Broker/Stream Congestion
* **Mechanism:** **Load Shedding** in the Go Ingestor. If Kafka partitions are saturated, the service drops low-priority system logs to prioritize medical telemetry.
* **Protection:** **Dead Letter Queues (DLQ)** for failed alert deliveries.

### C. gRPC Service (Java) Unavailability
* **Mechanism:** **Circuit Breaker** implementation in the BFF. It prevents "cascading failures" by stopping requests to the Java module if it's down, returning a cached or "Temporarily Unavailable" response.

### D. MQTT Broker Downtime
* **Mechanism:** **Local Device Buffering**. The pet collar stores data in internal Flash memory and uses a controlled **Uptake Strategy** upon reconnection to prevent a request storm.

## 4. Summary
By separating communication into **Ephemeral** (real-time stream) and **Durable** (critical alerts and Kafka logs), we balance extreme performance with medical-grade reliability.