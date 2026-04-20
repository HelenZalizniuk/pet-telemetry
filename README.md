# pet-telemetry
### High-Throughput IoT Monitoring System for Pet Health
> **Status:** Work in Progress. Architecture design phase.

[![Tech Stack](https://img.shields.io/badge/Stack-Go%20%7C%20Java%20%7C%20Kafka%20%7C%20Postgres-blue)](#)
[![Architecture](https://img.shields.io/badge/Architecture-EDA%20%7C%20C4%20Model-orange)](#)

## 📌 Overview
**pet-telemetry** is a high-performance backend ecosystem designed to ingest, process, and monitor real-time health data from thousands of IoT-enabled pet collars. 

This project demonstrates a transition from a traditional monolithic approach to a scalable **Event-Driven Architecture (EDA)**, focusing on high-write workloads, data consistency, and system observability.

## 🚀 Key Architectural Features
* **High-Throughput Ingestion:** Built with **Go** and a customized **Worker Pool** to handle 2000+ RPS.
* **Time-Series Optimization:** Utilizing **TimescaleDB** for efficient storage of heart rate and temperature metrics.
* **Reliable Messaging:** **Apache Kafka** acts as the central Message Bus, ensuring temporal decoupling.
* **Guaranteed Delivery:** Implementation of the **Transactional Outbox Pattern** to prevent data loss between the database and the message broker.
* **Scalability:** Horizontal sharding strategy by `pet_id`.

## 🏗 Architecture (C4 Model)

I used the **C4 Model** to document the system's architecture, moving from high-level context to container-level details.

### 1. System Context Diagram
Shows how **pet-telemetry** interacts with Pet Owners, Veterinarians, and IoT devices.
![System Context Diagram](./docs/diagrams/context.png)

### 2. Container Diagram
Detailed view of the tech stack: Go Ingestor, Java Medical Service, Kafka, and specialized Databases.
![Container Diagram](./docs/diagrams/container.png)