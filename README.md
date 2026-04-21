# pet-telemetry
### High-Throughput IoT Monitoring System for Pet Health

[![Tech Stack](https://img.shields.io/badge/Stack-Go%20%7C%20Java%20%7C%20Kafka%20%7C%20Postgres-blue)](#)
[![Architecture](https://img.shields.io/badge/Architecture-EDA%20%7C%20C4%20Model-orange)](#)

> **Status:** Work in Progress. Architecture design phase.

## 📌 Overview
**pet-telemetry** is a high-performance backend ecosystem designed to ingest, process, and monitor real-time health data from thousands of IoT-enabled pet collars. 

This project demonstrates a transition from a traditional monolithic approach to a scalable **Event-Driven Architecture (EDA)**, focusing on high-write workloads, data consistency, and system observability.

## 🚀 Key Architectural Features
* **High-Throughput Ingestion:** Built with **Go** and a customized **Worker Pool** to handle 2000+ RPS.
* **Time-Series Optimization:** Utilizing **TimescaleDB** for efficient storage of heart rate and temperature metrics.
* **Reliable Messaging:** **Apache Kafka** acts as the central Message Bus, ensuring temporal decoupling.
* **Guaranteed Delivery:** Implementation of the **Transactional Outbox Pattern** to prevent data loss between the database and the message broker.
* **Scalability:** Horizontal sharding strategy by `pet_id`.

---

## 🏗 Architecture (C4 Model)

I used the **C4 Model** to document the system's architecture, moving from high-level context to container-level details.

### 1. System Context Diagram
Shows how **pet-telemetry** interacts with Pet Owners, Veterinarians, and IoT devices.
![System Context Diagram](./docs/diagrams/context.png)

### 2. Container Diagram
Detailed view of the tech stack: Go Ingestor, Java Medical Service, Kafka, and specialized Databases.
![Container Diagram](./docs/diagrams/container.png)

---

## 📊 Data Strategy & Storage

The system uses a hybrid storage approach to satisfy both relational integrity and high-write performance requirements.

### 1. Storage Choice: TimescaleDB
* **Why:** Standard PostgreSQL struggles with massive time-series ingestion as indexes grow. TimescaleDB solves this via **Hypertables**, automatically partitioning data by time.
* **Optimization:** We use a **7-day chunk interval** and **Compression Policy** for data older than 30 days to reduce storage costs by up to 90%.

### 2. Database Schema (Key Entities)

#### `vitals_data` (Hypertable)
| Column | Type | Description |
| :--- | :--- | :--- |
| `time` | TIMESTAMPTZ | Event timestamp (Partitioning key) |
| `pet_id` | UUID | Foreign key to Pet Profile (Sharding key) |
| `heart_rate` | SMALLINT | BPM data |
| `temperature` | NUMERIC(4,2) | Celsius data |

---

## ⚙️ Scalability & Reliability (PACELC)

In designing the system, I applied the **PACELC** theorem to balance trade-offs:
* **Partition (P):** During a network partition, the system prioritizes **Availability (A)**. IoT collars can buffer data locally, and the Ingestor continues to accept heartbeat packets.
* **Else (E):** In normal operation, we prioritize **Latency (L)** over strict consistency for telemetry history, using asynchronous processing via Kafka.

### Infrastructure Strategy
* **Deployment:** Containerized via **Docker**, orchestrated by **Kubernetes**.
* **Auto-scaling:** Horizontal Pod Autoscaler (HPA) triggers new **Go Ingestor** instances based on CPU/RAM saturation or custom Kafka consumer lag metrics.
* **Fault Tolerance:** Multi-AZ deployment of Kafka brokers and PostgreSQL replicas.

## 📈 Observability & Monitoring
To maintain the **99.9% SLO**, the following stack is proposed:
* **Prometheus:** Scrapes Go-specific metrics (`goroutines_count`, `worker_pool_saturation`).
* **Grafana:** Dashboards for "Golden Signals" (Latency, Traffic, Errors, Saturation).
* **Alertmanager:** P0 alerts for Service Down and P1 alerts for abnormal health patterns.

---

## 🛠 Getting Started

To spin up the infrastructure (Databases, Message Broker, Cache), run:

```bash
cd deploy
docker-compose up -d

The system will be available with the following ports:

TimescaleDB: 5432

Kafka: 9092

Redis: 6379