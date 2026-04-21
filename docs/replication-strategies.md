# Replication Strategies: Leader-Based vs. Leaderless

This document analyzes the replication mechanisms used in the PetHealth Monitor to balance Data Consistency and System Availability under high load.

## 1. Leader-Based Replication (Standard Persistence)
Used for: **Pet CRM, Billing, and Medical Records (PostgreSQL).**

### How it Works:
* **Primary (Leader):** All **WRITE** operations are directed here. The leader records changes in its Write-Ahead Log (WAL).
* **Followers (Replicas):** Receive updates from the leader and serve **READ** requests.

### Architectural Impact:
* **Consistency:** High. By reading from the Leader, we ensure the most up-to-date information (Strong Consistency).
* **Availability:** Vulnerable. If the Leader fails, writes are blocked until a new leader is elected (Failover).
* **Scalability:** Excellent for read-heavy workloads. We scale by adding more followers.

### Scenario in PetHealth:
Ideal for the **Java Medical Service**. When a vet updates a pet's surgery record, we need a strict order of operations and no write conflicts.

---

## 2. Leaderless Replication (High-Velocity Streams)
Used for: **Raw Telemetry Storage (Cassandra/ScyllaDB style).**

### How it Works:
* **Quorum Model:** The client sends write requests to multiple nodes simultaneously.
* **Consistency Formula:** $W + R > V$ (where $W$ = write nodes, $R$ = read nodes, $V$ = total nodes).
* **Conflict Resolution:** Uses mechanisms like **Read Repair** and **LWW (Last Write Wins)** to merge data.

### Architectural Impact:
* **Consistency:** Eventual Consistency. Different nodes might briefly hold different versions of a heartbeat record.
* **Availability:** Maximum. As long as a quorum of nodes is online, the system accepts writes. There is no "Single Point of Failure."
* **Scalability:** Horizontally scales write throughput by adding more nodes to the cluster.

### Scenario in PetHealth:
Ideal for the **Go Ingestor** telemetry flow. With 2,000+ RPS, we cannot afford a single second of downtime. If one database node fails, the "Leaderless" cluster continues to absorb the stream of heart rate data without interruption.

---

## 3. Comparative Summary

| Feature | Leader-Based (PostgreSQL) | Leaderless (Telemetry DB) |
| :--- | :--- | :--- |
| **Write Point** | Single Node (Leader) | Any Node (Quorum) |
| **Logic Complexity** | Low (Sequential order) | High (Conflict resolution) |
| **Write Scaling** | Vertical (Limited by one node) | Horizontal (Add more nodes) |
| **Failover Reaction** | Requires election (Down-time) | No interruption (Self-healing) |
| **Best For** | Financials, CRM, Accounts | IoT Streams, High-load Ingestion |

## 4. Final Strategic Choice
For **PetHealth Monitor**, we implement a **Hybrid Replication Model**:
1.  **Leader-Based (Postgres):** For critical metadata where consistency and transactional integrity are non-negotiable.
2.  **Leaderless (LSM-based storage):** For the massive influx of telemetry where write-availability and partition tolerance are the top priorities (aligning with our **PA/EL** strategy).