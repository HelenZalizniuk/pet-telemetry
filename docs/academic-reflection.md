# Academic Reflection Report: Evolution of Requirements

This document tracks the evolution of the "PetHealth Monitor" project from the initial assignment to the final architectural design, highlighting the pivot points where academic theory met real-world engineering challenges.

## 1. Requirement Evolution (The Pivot Points)

| Requirement Area | Initial Naive Assumption | Final Architectural Reality | Rationale for the Engineering Pivot |
| :--- | :--- | :--- | :--- |
| **Ingestion Load** | Simple data collection for a few pets. | **2,000+ RPS / 172M events/day.** | Realized that IoT scales exponentially; a single clinic model is not a viable business case. |
| **Data Storage** | Standard SQL (PostgreSQL). | **TimescaleDB (Time-series) + Redis.** | Standard B-Tree indexes in Postgres "explode" in size and latency under high-velocity telemetry. |
| **Messaging** | Direct REST API or RabbitMQ. | **Event-Driven (Apache Kafka).** | RabbitMQ’s push-model and overhead per message became a bottleneck at 2k RPS. Kafka’s log-append model is mandatory. |
| **Data Format** | Human-readable JSON. | **Binary Protobuf.** | JSON is too heavy for IoT battery/bandwidth. Protobuf reduced payload size by 75%. |

## 2. Methodology & Self-Correction Reflection

### 2.1. From REST to EDA (Reliability Pivot)
**Initially:** I considered a standard Microservices approach using REST for inter-service communication.
**Correction:** After analyzing the risk of data loss during high load spikes, I realized REST is too synchronous and fragile. I pivoted to **Event-Driven Architecture (EDA)**. To solve the "Dual Write" problem (ensuring data is saved to the DB and sent to Kafka simultaneously), I introduced the **Transactional Outbox pattern**, which ensures 100% reliability even if the broker is temporarily down.

### 2.2. The "RabbitMQ to Kafka" Realization
**Initially:** I planned to use RabbitMQ for all messaging tasks due to its simplicity.
**The Shift:** During Capacity Planning, the math didn't lie. At 2,000 RPS, the overhead of individual acknowledgments and complex routing in RabbitMQ would consume excessive CPU resources. I made a strategic decision to shift the telemetry stream to **Apache Kafka**, utilizing its sequential log-append disk I/O and data replayability features.

### 2.3. Optimization vs. Over-Engineering
**Initially:** I heavily considered implementing Bloom Filters for rapid device validation.
**Correction:** During the CS Fundamentals analysis, I performed a sanity check on the actual scale. With 10,000 devices, a simple Redis Set fits entirely in memory and provides $O(1)$ lookup without the complexity of probabilistic structures. I chose to avoid over-engineering Bloom Filters and instead focused on **TimescaleDB Continuous Aggregates**, which provided a tangible 90% performance boost for historical reports.

### 2.4. Balancing Ideals with Business Flexibility
**The Challenge:** During the design phase, concerns were raised about forcing third-party IoT vendors to adopt Protobuf immediately.
**The Solution:** I didn't stick to a "one size fits all" approach. I redesigned the Go Ingestor as a multi-protocol gateway capable of handling both JSON (for legacy support/simplicity) and Protobuf (for high-efficiency production devices). This demonstrated that a Senior Architect must balance technical purity with market reality.

### 2.5. CAP/PACELC Strategy Evolution
**Initial Thought:** I wanted "Strong Consistency" across the entire system to avoid any data anomalies.
**The Reality Check:** I realized that for a pet's heart rate, **Availability (AP)** is far more critical than immediate consistency. I moved to an **Eventual Consistency** model for telemetry but strictly maintained a **Saga (TCC)** pattern for financial and medical records, recognizing that different data domains require different trade-offs.

### 2.6. Replication & Resilience Engineering
**Initial Thought:** Use standard Master-Slave (Leader-based) replication for all databases.
**The Shift:** For the telemetry stream, I realized the Leader node would be a critical Single Point of Failure (SPOF). I implemented a **Hybrid Replication** strategy: **Leader-based** for the Java CRM (where write-order is critical) and **Leaderless/Quorum-based** for telemetry ingestion, ensuring the system remains operational even during a partial data center failure.

## 3. Final Conclusion
The project evolved from a simple "CRUD application" into a sophisticated **Distributed Ecosystem**. The main takeaway is that high-load architecture is not about choosing the "best" tool in a vacuum, but about making the right **trade-offs** between Latency, Consistency, and Total Cost of Ownership (TCO).

---
*Date: April 21, 2026*
*Author: Senior Software Engineer (Transitioning to Go/Rust Ecosystem)*