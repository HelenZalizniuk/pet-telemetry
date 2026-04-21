# System Decomposition & Scaling in Depth

This document details the strategy for splitting the monolithic architecture into specialized services and the rationale behind the chosen decomposition patterns.

## 1. Scaling in Depth (Internal Optimization)
Instead of simply adding more servers, we maximize the efficiency of core nodes:

* **Go Ingestor (Worker Pool):** We strictly limit the number of concurrent goroutines. 
    * *Rationale:* This prevents memory thrashing and ensures predictable performance during traffic spikes (2000+ RPS). It allows us to fully saturate CPU cores without context-switching overhead.
* **TimescaleDB (Hypertables):** Automatic time-based partitioning (chunking).
    * *Rationale:* Ensures that new telemetry data always hits "hot" chunks that reside in RAM, maintaining high write speeds.

## 2. Decomposition Strategy
We applied a hybrid approach to decouple the system:

### A. By Workload (The "Gatekeeper" Pattern)
* **Component:** Go Ingestor.
* **Rationale:** Telemetry ingestion is a high-frequency, low-logic task. Separating it allows us to scale the "entry point" independently from the UI and business logic layers.

### B. By Transactions (The "Source of Truth" Pattern)
* **Component:** Java Medical Service.
* **Rationale:** Device registration and pet-owner linking require **ACID compliance**. We use Java's mature ecosystem to ensure transactional integrity—preventing orphaned devices or duplicated medical records.

## 3. Architectural Variants Considered

| Approach | Components | Rationale |
| :--- | :--- | :--- |
| **Option 1: Coarse-Grained (Selected)** | **Ingestion Service (Go)** & **Medical Registry (Java)** | **Best for MVP.** Minimizes network overhead (latency) while isolating the high-load ingestion path. |
| **Option 2: Fine-Grained** | **Ingestion**, **Device Registry**, **Alerts & Analytics** | **High Complexity.** Best for large teams. Ensures that an outage in the Alerting API doesn't block device registration. |

## 4. Applied Design Patterns

* **Database Per Service:** Ensures autonomy. The Go team can optimize telemetry storage (TimescaleDB) without impacting the relational PostgreSQL tables managed by the Java team.
* **BFF (Backend for Frontend):** Provides a unified API for mobile users. It aggregates pet metadata from Java and real-time vitals from Redis.
* **Eventual Consistency:** Solved via Kafka. When a pet is deleted in the Java Registry, an asynchronous event tells the Go Ingestor to stop accepting data from that device.

## Conclusion
The final architecture separates the **Fast Stream Layer (Go)** from the **Reliable Transaction Layer (Java)**. This balances performance (Scaling in Depth) with data integrity (ACID).