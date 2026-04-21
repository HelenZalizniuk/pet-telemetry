# Academic Reflection Report: Evolution of Requirements

This document tracks the evolution of the "PetHealth Monitor" project from the initial assignment (Task-1) to the final architectural design.

## 1. Requirement Evolution (Before vs. After)

| Requirement Area | Initial State (Task-1) | Final Architectural Design | Rationale for Change |
| :--- | :--- | :--- | :--- |
| **Ingestion Load** | Basic data collection | **2000+ RPS / High-Load** | Initial requirements underestimated the scale of concurrent IoT connections. |
| **Data Storage** | Standard SQL Database | **TimescaleDB (Time-series)** | Standard RDBMS indexes fail under high-write telemetry workloads. |
| **Messaging** | Direct API calls | **Event-Driven (Kafka)** | Decoupling is necessary for system resilience and async alerting. |

## 2. Methodology Reflection
The initial list of questions for stakeholders focused on basic functionality. The finalized "Senior-level" questions focus on **System Boundaries, Data Lifecycle, and Cost of Ownership**, which led to a more robust and production-ready architecture.

**I initially considered a standard Microservices approach with REST. However, after analyzing the 2,000 RPS requirement and the risk of data loss, I pivoted to EDA (Event-Driven Architecture) and added the Transactional Outbox pattern to ensure reliability.**

I initially calculated storage based on standard SQL row sizes. However, for the final design, I recalculated it for Protobuf and factored in TimescaleDB compression, which significantly reduced the projected TCO (Total Cost of Ownership).

I moved beyond simple 'health checks'. I defined specific **SLIs/SLOs** that directly correlate with pet safety (e.g., Data Durability 100%). I also integrated the **Golden Signals** approach into the alerting strategy, which is a standard for professional high-load systems. I replaced generic Hit Rate metrics with DB Lookup Rate to better reflect system bottlenecks and introduced a graded alerting system for device connectivity (from 1 min to 48 hours) to balance technical monitoring with business needs.



