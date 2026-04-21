# Technical Specifications for Distributed Storage

This document details the low-level technical requirements for the PetHealth Monitor storage layer, focusing on replication, sharding, and consensus mechanisms.

## 1. Replication Strategy
* **Choice:** **Single-Leader Replication** (Primary with asynchronous followers).
* **Rationale:** * **Order of Operations:** For medical records and prescriptions, the sequence of events is vital. A single leader ensures writes are applied in the exact order they were received.
    * **High Availability:** If the primary node fails, an automated failover promotes a follower to leader.
    * **Read Scaling:** Allows veterinarians to query historical health data from replicas without impacting the write-performance of the primary node.

## 2. Partitioning (Sharding)
* **Choice:** **Horizontal Sharding** based on `pet_id`.
* **Method:** **Hash-based Partitioning**.
* **Rationale:**
    * **Scalability:** As the number of active pet collars grows, telemetry volume will exceed the Disk I/O limits of a single server.
    * **Hotspot Prevention:** Hashing the `pet_id` ensures an even distribution of data across all shards, preventing a single server from becoming a bottleneck while others remain idle.

## 3. Transaction Model & Isolation
* **Choice:** **ACID** within local shards; **Saga Pattern** for cross-service operations.
* **Isolation Level:** **Read Committed**.
* **Rationale:**
    * **ACID Integrity:** Critical for billing and appointment scheduling to prevent data loss or double-booking within a single shard.
    * **Performance Balance:** *Read Committed* is the "Golden Mean"—it prevents dirty reads while maintaining high throughput, which is essential for our 2,000+ RPS target.
    * **Saga over 2PC:** Distributed 2-Phase Commits are too slow for high-load systems. We use Sagas with compensating transactions for multi-service flows (e.g., Payment -> Appointment Booking).

## 4. Data Integrity & Consistency
* **Choice:** **Eventual Consistency** for telemetry; **Strong Consistency** for financials.
* **Rationale:**
    * **Telemetry:** If a heart rate update (80 BPM -> 82 BPM) takes 200ms to propagate to all replicas, the impact is negligible. Availability is prioritized.
    * **Read-Your-Writes:** For clinical entries, we ensure the vet immediately sees the prescription they just saved, providing a consistent user experience.

## 5. Consensus Mechanism
* **Choice:** **Raft Algorithm** (via Etcd or Consul).
* **Role in System:**
    * **Leader Election:** Raft ensures that if the database primary or a Kubernetes service fails, the cluster nodes reach a consensus to elect exactly one new leader.
    * **Split-Brain Prevention:** Guarantees that two nodes never act as "Leader" simultaneously, which would otherwise lead to catastrophic data corruption.