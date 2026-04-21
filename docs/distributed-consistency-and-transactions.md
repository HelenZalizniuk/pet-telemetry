# Distributed Consistency & Reliable Transactions

This document analyzes the reliability guarantees of the system, justifies the choice of BASE over ACID for telemetry, and describes the algorithm for handling cross-service financial operations.

## 1. ACID Analysis: The Trade-off of Isolation

In modern high-load systems, **Isolation (I)** is often considered the most "expendable" or least weighty requirement of ACID.

* **Rationale:** Achieving the highest isolation level (**Serializable**) requires aggressive locking mechanisms. In a system handling 2,000+ RPS, this creates massive contention, killing throughput and turning the system into a bottleneck.
* **Industry Standard:** We prioritize performance by using **Read Committed** (the default in PostgreSQL). Any potential anomalies (like Non-repeatable reads) are handled via **Optimistic Locking (Versioning)** or within the application business logic.

## 2. Consistency Model: ACID vs. BASE

For the **PetHealth Monitor** core (vitals ingestion), we have selected the **BASE** model.

* **Basically Available:** It is more important to accept a heart rate packet from a collar "at any cost" than to ensure instant consistency across all global replicas.
* **Soft State & Eventual Consistency:** A pet's vitals in the owner's app might update with a 1-second delay as data flows through Kafka. This latency is negligible for health monitoring but essential for scaling to millions of devices.
* **Exception:** For the **Billing/Payment Module**, we maintain **ACID** properties within the local database context while using **Saga patterns** for cross-service coordination.

## 3. Reliable Inter-Service Transactions (The Banking Pattern)

When transferring assets (e.g., funds or credits) between independent entities (Bank A to Bank B), we cannot use a simple 2-Phase Commit (2PC). Instead, we implement a **Saga Pattern (TCC - Try-Confirm-Cancel)** with **Idempotency**.

### Required API Specifications:
1.  `POST /reserve`: (**Try**) Freezes the amount. Requires an `idempotency_key`.
2.  `POST /confirm`: (**Confirm**) Finalizes the debit/credit.
3.  `POST /cancel`: (**Cancel**) Releases the frozen funds in case of failure.

### Reliability Algorithm:
1.  **Transaction ID Generation:** Create a unique UUID for the entire flow.
2.  **Step 1 (Bank A - Reserve):** Call `/reserve`. If it fails, stop immediately (funds remain safe).
3.  **Step 2 (Local Outbox):** Save the state "Reserve successful, pending Bank B" to the local database.
4.  **Step 3 (Bank B - Deposit):** Call `/confirm` in Bank B using the `idempotency_key`. 
    * *Failure Handling:* Retry with **Exponential Backoff**. The `idempotency_key` ensures the pet owner isn't charged twice.
5.  **Step 4 (Bank A - Commit):** Once Bank B confirms, call `/confirm` in Bank A to finalize the deduction.
6.  **Compensation:** If Bank B returns a hard error (e.g., account blocked), call `/cancel` in Bank A to unfreeze the funds.

## 4. Resilience to Component Failure

* **Intermediary App Crash:** Upon restart, a **Transactional Relay** reads the **Outbox** table and resumes unfinished transactions from the last known state.
* **Network/Bank Downtime:** Retries with the same idempotency key guarantee "At-least-once" delivery without the risk of double-processing.