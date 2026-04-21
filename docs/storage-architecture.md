# Storage Architecture & Data Modeling

This document outlines the multi-tiered storage strategy designed to balance ACID compliance for business data with high-performance ingestion for telemetry.

## 1. Storage Tiering Logic

We implement a four-layer storage hierarchy to optimize for speed, cost, and reliability:

| Tier | Technology | Role | Data Type |
| :--- | :--- | :--- | :--- |
| **L1: In-Memory (Local)** | **Go LRU Cache** | "Reflexes" | Recent thresholds for immediate validation without network calls. |
| **L2: Shared Cache** | **Redis** | "Active Brain" | Real-time device statuses, alert thresholds, and Pub/Sub for config sync. |
| **L3: Time-Series** | **TimescaleDB** | "Deep Memory" | Massive streams of heart rate and temperature data (Hypertables). |
| **L4: Relational** | **PostgreSQL** | "Foundation" | Business logic: User profiles, Pet-to-Device links, Billing, and Metadata. |

## 2. Technology Rationale (Trade-offs)

### Why TimescaleDB over Cassandra or MongoDB?
* **vs. MongoDB:** Document-oriented storage is too verbose for 172M daily points. TimescaleDB’s column-based compression is **10-20x more efficient** for repetitive telemetry.
* **vs. Cassandra:** While horizontally scalable, Cassandra requires "Query-first" design. TimescaleDB allows us to use standard **SQL for complex medical analytics**.
* **Benefit:** **Continuous Aggregates** pre-calculate 1-hour summaries, making UI graphs load instantly.

### Why Redis over Memcached?
* **Data Structures:** We need more than just strings. Redis **Hashes** store device profiles, and **Pub/Sub** ensures the Go Ingestor is instantly notified when a vet updates a pet's thresholds.

## 3. Data Schema & Structures

### A. Redis (Operational Data)
*Key:* `device_profile:{device_id}` (Type: **Hash**)
* `owner_id`: UUID, `min_heart_rate`: Integer, `max_heart_rate`: Integer, `status`: Enum, `last_sync`: Timestamp.

### B. TimescaleDB (Telemetry Hypertable)
| Column | Type | Description |
| :--- | :--- | :--- |
| **time** | TIMESTAMPTZ | Event time (**Primary partition key**) |
| **device_id** | UUID | Collar ID (**Secondary indexing/sharding**) |
| **heart_rate** | SMALLINT | Pulse (Smallint for space efficiency) |
| **temperature** | NUMERIC(4,2) | Temperature with 2-decimal precision |
| **activity_level**| TINYINT | Activity score (0-5) |

### C. PostgreSQL (Business Metadata)
Standard relational structure ensuring **referential integrity**.
* **Users:** `id, email, password_hash, created_at`.
* **Pets:** `id, user_id (FK), name, breed, birth_date`.
* **Devices:** `id, pet_id (FK), serial_number, firmware_version`.

## 4. Technical Requirements & Integrity

* **Transactions:** * **Relational:** Full ACID transactions for Users/Pets to prevent orphaned records.
    * **Telemetry:** Batched inserts into TimescaleDB chunks to ensure atomicity at scale without locking the table.
* **Integrity:** Strict Foreign Key constraints in PostgreSQL. Data validation in Go Ingestor before writing to TimescaleDB.
* **Replication:** * Multi-AZ Synchronous replication for PostgreSQL (Zero Data Loss).
    * Asynchronous replication for TimescaleDB to maintain high ingestion throughput.

## 5. Storage Analysis (PACELC/CAP)
* **PostgreSQL:** Optimized for **Consistency** (PC/EC) for financial and profile data.
* **TimescaleDB:** Optimized for **Availability** (PA/EL) during ingestion, ensuring telemetry is stored even during minor cluster rebalancing.
* **Partitioning:** TimescaleDB **Hypertables** automatically partition data by time chunks (1-day intervals), enabling high-speed data retention and index management.