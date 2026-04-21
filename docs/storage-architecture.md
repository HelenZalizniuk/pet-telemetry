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
* **vs. Cassandra:** While horizontally scalable, Cassandra requires "Query-first" design. TimescaleDB allows us to use standard **SQL for complex medical analytics** (e.g., correlating breed vs. average heart rate).
* **Benefit:** **Continuous Aggregates** pre-calculate 1-hour summaries, making UI graphs load instantly.

### Why Redis over Memcached?
* **Data Structures:** We need more than just strings. Redis **Hashes** store device profiles, and **Pub/Sub** ensures the Go Ingestor is instantly notified when a vet updates a pet's thresholds in the Java service.

## 3. Data Schema & Structures

### A. Redis (Operational Data)
*Key:* `device_profile:{device_id}` (Type: **Hash**)
* `owner_id`: UUID
* `min_heart_rate`: Integer (Low threshold)
* `max_heart_rate`: Integer (High threshold)
* `status`: Enum (active, charging, offline)
* `last_sync`: Timestamp

### B. TimescaleDB (Telemetry Hypertable)
Optimized for high-speed sequential writes.

| Column | Type | Description |
| :--- | :--- | :--- |
| **time** | TIMESTAMPTZ | Event time (Primary partition index) |
| **device_id** | UUID | Collar ID (Secondary index for segmentation) |
| **heart_rate** | SMALLINT | Pulse (Using SMALLINT to save disk space) |
| **temperature** | NUMERIC(4,2) | Temperature with 2-decimal precision |
| **activity_level**| TINYINT | Activity score (0-5) |

### C. PostgreSQL (Business Metadata)
Standard relational structure ensuring **referential integrity**.
* **Users:** `id, email, password_hash, created_at`.
* **Pets:** `id, user_id (FK), name, breed, birth_date`.
* **Devices:** `id, pet_id (FK), serial_number, firmware_version`.

## 4. Final Trade-off Summary
* **PostgreSQL:** Chosen for **ACID guarantees** where relational integrity is non-negotiable.
* **TimescaleDB:** Chosen for **Delta-delta compression** and automated chunking, reducing storage costs by 90%.
* **Redis:** Chosen for **sub-1ms latency** and inter-service synchronization.