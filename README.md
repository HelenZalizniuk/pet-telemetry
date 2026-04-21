# 🐾 pet-telemetry: High-Load IoT Monitoring System

[![Tech Stack](https://img.shields.io/badge/Stack-Go%20%7C%20Java%20%7C%20Kafka%20%7C%20Postgres-blue)](#)
[![Architecture](https://img.shields.io/badge/Architecture-EDA%20%7C%20C4%20Model-orange)](#)
[![Theory](https://img.shields.io/badge/Design-PA%2FEL%20%7C%20Saga%20%7C%20Raft-green)](#)

## 📌 Overview
**pet-telemetry** is a high-performance backend ecosystem designed to ingest, process, and analyze real-time health data (heart rate, temperature, activity) from thousands of IoT-enabled pet collars.

This project demonstrates a strategic transition from a monolithic approach to a scalable **Event-Driven Architecture (EDA)** capable of handling **2,000+ RPS (172M+ events per day)**, focusing on high-write workloads, fault tolerance, and deep observability.

---

## 🏗 Architecture (C4 Model)
I used the **C4 Model** to document the system's architecture, moving from high-level context to container-level details. This ensures both business stakeholders and engineers understand the data flow.

### 1. System Context Diagram
Describes how **pet-telemetry** interacts with Pet Owners, Veterinarians, and IoT devices.  
![System Context Diagram](./docs/diagrams/context.png)

### 2. Container Diagram
A detailed view of the tech stack: Go Ingestor, Java Medical Service, Kafka, and specialized Databases (Tiered Storage).  
![Container Diagram](./docs/diagrams/container.png)

---

## 📄 Detailed Documentation Index
I have meticulously documented every architectural decision. It is recommended to explore them in the following order:

1. [**Architectural Vision**](./docs/architecture-deep-dive.md) – Project vision, business goals, and the Java-to-Go pivot.
2. [**Architectural Style**](./docs/architectural-style.md) – Hybrid decomposition and system patterns.
3. [**Capacity Planning**](./docs/capacity-planning.md) – Throughput calculations for 2,000+ RPS.
4. [**Scalability Analysis**](./docs/scalability-analysis.md) – Vertical, horizontal, and "in-depth" scaling strategies.
5. [**Decomposition Strategy**](./docs/decomposition-strategy.md) – Logic behind service split and communication boundaries.
6. [**Data Access Strategy**](./docs/data-access-strategy.md) – Protocols (gRPC, MQTT, Kafka) and communication flows.
7. [**Data Serialization & Storage**](./docs/data-serialization-storage.md) – Protobuf vs. JSON for binary efficiency.
8. [**Storage Architecture**](./docs/storage-architecture.md) – Multi-tiered design with Redis, PostgreSQL, and TimescaleDB.
9. [**Observability**](./docs/observability.md) – Monitoring stack, SLOs (99.9%), and health-check logic.
10. [**CS Fundamentals**](./docs/data-structures-and-memory.md) – LSM-Trees, B-Trees, and memory management.
11. [**CAP & PACELC Analysis**](./docs/cap-pacelc-analysis.md) – Distributed systems theory and the PA/EL strategy.
12. [**Analytics & Reporting**](./docs/analytics-and-reporting.md) – Streaming vs. Batching design for medical insights.
13. [**Distributed Consistency**](./docs/distributed-consistency-and-transactions.md) – Saga patterns, ACID vs. BASE, and reliability.
14. [**Replication Strategies**](./docs/replication-strategies.md) – Comparison of Leader-based vs. Leaderless replication.
15. [**Distributed Storage Specs**](./docs/distributed-storage-specs.md) – Sharding, consensus (Raft), and final technical specs.
16. [**Academic Reflection**](./docs/academic-reflection.md) – Project summary and key design takeaways.

---

## 🏗 Data Contracts & Serialization
To ensure seamless communication between the **Go Ingestor** and **Java Medical Service**, we use **Protocol Buffers (Protobuf)**.

* **Contract Definition:** [telemetry.proto](./api/telemetry.proto)
* **Key Benefits:** * **75% smaller payloads** compared to standard JSON.
    * **Automatic Code Generation** for both Golang and Java.
    * **Type Safety:** Ensures valid data types for heart rate and activity levels.

---

## 🚀 Key Architectural Features
* **High-Throughput Ingestion:** Custom **Go Worker Pool** for non-blocking telemetry processing.
* **Time-Series Optimization:** Metrics stored in **TimescaleDB** using Hypertables (90% storage savings).
* **Reliable Messaging:** **Apache Kafka** acts as the central message bus and buffer.
* **Binary Efficiency:** **Protobuf** integration to extend IoT device battery life.
* **Hybrid Replication:** Leader-based for CRM data; Quorum-based for telemetry.

---

## 📊 Data Strategy & Distributed Theory
* **PACELC (PA/EL):** Prioritizes **Availability** during partitions and **Latency** in normal operations.
* **Consensus:** **Raft Algorithm** for Leader Election to prevent Split-Brain scenarios.
* **Sharding:** **Horizontal Hash-based Sharding** by `pet_id`.

## 📈 Observability (SLO 99.9%)
* **Prometheus & Grafana:** Monitoring "Golden Signals" and Go-specific metrics.
* **ELK Stack:** Centralized log aggregation.
* **Alerting:** Graded alerting system via Alertmanager.

---

## 🛠 Getting Started
```bash
cd deploy
docker-compose up -d

Infrastructure Map:
TimescaleDB: 5432 (Telemetry History)

PostgreSQL: 5433 (Business/CRM Data)

Kafka: 9092 (Message Bus)

Redis: 6379 (Live Cache & Pub/Sub)

Developed as part of the Advanced Software Architecture Program, 2026.