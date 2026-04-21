# Scalability Analysis: Vertical vs. Horizontal

In this section, I evaluate how the **pet-telemetry** system handles growth, comparing Vertical and Horizontal scaling strategies for its core components.

## 1. Vertical Scaling (Scaling Up) Analysis
*Increasing resources (CPU, RAM) of existing nodes.*

| Component | Effectiveness | Limitations |
| :--- | :--- | :--- |
| **Go Ingestor** | **Medium.** Go handles multi-core systems well, but a single node becomes a Single Point of Failure (SPOF). | Hard limit on physical hardware; risk of complete downtime during maintenance. |
| **TimescaleDB** | **High (Initial).** More RAM significantly speeds up index lookups and time-series chunk management. | Extremely expensive for high-end instances; eventually hits a write-lock bottleneck. |

## 2. Horizontal Scaling (Scaling Out) Analysis
*Adding more instances of the service or database.*

| Component | Effectiveness | Implementation Strategy |
| :--- | :--- | :--- |
| **Go Ingestor** | **Maximum.** Services are stateless. | Using **Kubernetes HPA** to spin up new pods based on CPU/RAM saturation. |
| **Kafka Bus** | **Maximum.** High throughput is achieved by adding more partitions and brokers. | Horizontal expansion by increasing the partition count for `vitals.raw`. |
| **TimescaleDB** | **High.** Sharding telemetry data across multiple nodes. | **Multi-node TimescaleDB** or manual sharding by `pet_id` range. |

## 3. Scaling Decision & Rationale

For the **PetHealth Monitor**, I have chosen a **Hybrid Horizontal Scaling** approach:

1. **Service Layer (Go/Java):** 100% Horizontal. This allows the system to be "Elastic" — growing during peak morning/evening pet activity and shrinking at night to save costs.
2. **Data Layer (TimescaleDB):** Horizontal Sharding. Since we ingest ~25GB/day, a single server would reach its limit within months. Sharding by `pet_id` ensures that data for a single pet stays consistent on one node while the overall load is distributed.

## 4. Bottleneck Mitigation
To prevent scaling issues, the following optimizations are implemented:
* **Connection Pooling:** Using `pgbouncer` for the database to handle thousands of concurrent Go-worker connections.
* **Backpressure Handling:** Using Kafka as a buffer to prevent the Ingestor from overwhelming the Database during traffic spikes.