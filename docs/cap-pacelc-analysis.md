# Distributed Systems Theory: CAP & PACELC Analysis

This document outlines the theoretical foundation of the PetHealth Monitor's distributed architecture, justifying the trade-offs between consistency, availability, and latency.

## 1. CAP Theorem: Choosing AP (Availability + Partition Tolerance)

In a distributed IoT ecosystem, network instability is a given. We prioritize **AP** to ensure the system remains functional under stress.

* **P (Partition Tolerance):** Mandatory. Collars may lose connection, or the Go Ingestor might be isolated from the Java core. The system must continue collecting data locally or at available nodes.
* **A (Availability):** Critical. Pet owners and vets must always have access to the dashboard. Seeing data that is 10 seconds old is acceptable; seeing a "500 Internal Server Error" is not.
* **C (Consistency):** We accept **Eventual Consistency**. It is not vital for a heart rate measurement of "80 BPM" to appear in the analytical database the exact millisecond it's recorded, as long as it arrives reliably within a few hundred milliseconds.

## 2. PACELC Framework: PA/EL Strategy

The PACELC extension helps us define behavior during both failure and normal operation. Our formula is **PA/EL**.

### P + A (In case of Partition -> Availability)
If a network partition occurs, we choose **Availability**. The Go Ingestor's Worker Pool continues to accept telemetry and store it in local buffers or Kafka, even if the primary PostgreSQL/TimescaleDB is temporarily unreachable.

### E + L (Else -> Latency)
In normal operation (No Partition), we prioritize **Low Latency** over immediate Consistency. 
* **Rationale:** Processing 2,000+ RPS requires non-blocking operations. Waiting for synchronous write acknowledgments from all database replicas for every single heartbeat would clog the Worker Pool and cause the system to lag behind real-time.

## 3. Technical Implementation of PA/EL

How our technical choices in the Go Ingestor support these characteristics:

| Technical Solution | Supporting Attribute | Function |
| :--- | :--- | :--- |
| **Worker Pool** | **Availability & Latency** | Allows rapid ingestion of incoming packets. Even if the DB slows down, workers keep "swallowing" data. |
| **Outbox Pattern** | **Consistency & Partition Tolerance** | Ensures "at-least-once" delivery. Events are saved locally first, then pushed to Kafka, guaranteeing no data loss during network splits. |
| **Kafka Buffering** | **Eventual Consistency** | Acts as a shock absorber. Alert services can lag slightly behind ingestion but will always catch up without slowing down the input. |
| **TimescaleDB Chunks** | **Latency** | Writing only to the active "time-chunk" minimizes disk I/O contention and keeps write latency low. |

## 4. Summary
The **PA/EL** model is the most pragmatic choice for a medical monitoring system. By prioritizing availability and low latency, we ensure that the vitals of thousands of pets are tracked in real-time, with architectural safeguards ensuring that the data eventually reaches a consistent state across the entire hybrid (Java/Go) landscape.