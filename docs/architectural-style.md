# Architectural Style & Design Patterns

This document explains the rationale behind choosing the specific architectural style for the **pet-telemetry** ecosystem.

## 1. Evaluation of Architectural Styles

I analyzed three potential styles to solve the high-load telemetry problem:

| Style | Pros | Cons | Verdict |
| :--- | :--- | :--- | :--- |
| **Monolithic (Legacy)** | Simple deployment, shared database. | Performance bottleneck at 500+ RPS; tight coupling. | **Rejected** |
| **Microservices (REST)** | Better scaling, independent deployment. | "Synchronous hell"; cascading failures under high IoT load. | **Rejected** |
| **Event-Driven (EDA)** | **Asynchronous processing, extreme scalability, high resilience.** | Higher complexity in monitoring and debugging. | **Selected** |

## 2. Why Event-Driven Architecture (EDA)?

The transition to EDA was driven by the following factors:
* **Temporal Decoupling:** The Go Ingestor can accept data even if the Medical Service or Database is temporarily down.
* **Write-Heavy Ingestion:** Kafka acts as a massive buffer, protecting downstream services from traffic spikes.
* **Extensibility:** New features (e.g., ML-based health analysis) can be added as new consumers without changing existing code.

## 3. Core Design Patterns Applied

* **Transactional Outbox:** To ensure data consistency between the DB (TimescaleDB) and the Message Broker (Kafka). This prevents "lost events" if Kafka is unreachable.
* **Worker Pool (Go):** Efficiently managing system resources (CPU/RAM) by limiting concurrent goroutines during ingestion.
* **Sidecar/Relay:** A dedicated process for streaming outbox events, keeping the main business logic lean.