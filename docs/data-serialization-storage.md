# Data Strategy: Serialization & Storage Rationale

This document details the selection of data formats and storage engines, focusing on the trade-offs between IoT constraints, performance, and developer experience.

## 1. Serialization: Protobuf vs. JSON vs. MessagePack

For a high-load system (2,000+ RPS), minimizing payload size and CPU overhead is critical.

| Characteristic | JSON (Legacy/MVP) | MessagePack | Protocol Buffers (Target) |
| :--- | :--- | :--- | :--- |
| **Type** | Text (Verbose) | Binary (Schemaless) | Binary (Strict Schema) |
| **Efficiency** | Low (Heavy parsing) | Medium | **High (Fastest)** |
| **Traffic** | 100% (Baseline) | ~60% | **~25% (4x savings)** |
| **IoT Suitability** | Low (Battery drain) | Medium | **Excellent (Nanopb)** |
| **Flexibility** | High | High | Moderate (Requires code gen) |

### Why we chose Protocol Buffers (Protobuf)
1. **Battery Autonomy:** Smaller binary payloads reduce the active time of the collar's radio module, extending battery life by 15-20%.
2. **CPU Efficiency:** Go handles binary deserialization much faster than JSON parsing, reducing the number of servers needed to handle 2,000 RPS.
3. **Strict Contract:** Using `.proto` files ensures that the hardware team (firmware) and software team (backend) stay in sync, preventing runtime errors.

---

## 2. Transition Strategy: Multi-Protocol Support

To accommodate different hardware versions, the Go Ingestor implements the **Strategy Pattern** for decoding payloads:

* **Dual-Topic Ingress:**
    * `vitals/v1/json` — For legacy or third-party devices.
    * `vitals/v2/proto` — For our high-efficiency smart collars.
* **Decoder Interface:** The Go code uses a `PayloadDecoder` interface. The system detects the format from the topic/metadata and converts it into a unified **Internal DTO**.
* **Business Logic Neutrality:** The rest of the system (Kafka, Redis, Java) remains agnostic to the original transmission format.

---

## 3. Storage Strategy: Why TimescaleDB?

We use TimescaleDB (PostgreSQL-based) to handle 172 million telemetry points per day.

### A. Performance & Compression
* **Hypertables:** Data is automatically partitioned into "chunks" by time. This keeps the "active" write index in RAM, maintaining stable 2,000+ RPS.
* **Adaptive Compression:** Algorithms like *Delta-delta* and *Gorilla* allow us to compress historical data by up to **90%**, significantly reducing disk costs.

### B. Analytical Capabilities
* **Continuous Aggregates:** The database automatically pre-calculates 5-minute and 1-hour averages. When a vet opens a pet's dashboard, the graph loads instantly without scanning millions of raw rows.
* **Retention Policies:** We automatically drop raw data older than 6 months while keeping aggregated summaries, managing the storage lifecycle without manual intervention.

---

## 4. Key Questions for Stakeholders

If considering a pivot to JSON/MessagePack, we must evaluate:
1. **Target Battery Life:** Is a 15-20% reduction in autonomy acceptable if we use JSON?
2. **Connectivity Budget:** What is the SIM-card data limit for 50,000 devices? (Protobuf can save thousands of dollars/month).
3. **Hardware Constraints:** Does the chosen microcontroller have enough RAM for dynamic MsgPack parsing, or do we need the static memory footprints of **Nanopb**?