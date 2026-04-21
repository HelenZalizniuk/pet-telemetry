# Low-Level Design: Go Telemetry Ingestor

This document details the internal workings of the Go Ingestor, the most performance-critical component of the system. It describes the flow of data from the IoT device to the message broker.

## 1. Component Logic Diagram
The following diagram illustrates the internal components of the Go Ingestor and their interactions:

![Component Diagram](./diagrams/component.png)

---

## 2. Request Lifecycle (Step-by-Step)
1. **Ingress:** The service receives a binary payload via gRPC/HTTP.
2. **Fast Validation:** Middleware checks the `Device-ID` against a **Redis Set** (O(1)) to drop unauthorized traffic immediately.
3. **Unmarshalling:** The Protobuf payload is unmarshalled into a Go struct.
4. **Buffering:** The message is pushed into an internal **Buffered Channel**.
5. **Worker Processing:** A free worker from the **Worker Pool** picks up the message.
6. **Enrichment:** The worker adds a server-side timestamp and metadata.
7. **Async Write:** The message is handed over to the **Kafka Async Producer**.

---

## 3. Concurrency Model (Worker Pool)
To handle 2,000 RPS without spawning 2,000 goroutines per second (which would stress the Garbage Collector), we use a fixed-size Worker Pool.

### Key Logic:
* **Bounded Buffer:** We use a buffered Go channel to hold incoming requests.
* **Worker Reuse:** A pre-defined number of goroutines (50-100) listen on the channel, ensuring the system doesn't exceed CPU limits.

```go
// Simplified LLD Logic Implementation
func worker(id int, jobs <-chan TelemetryData) {
    for j := range jobs {
        // Step 6: Enrichment & Step 7: Kafka Produce
        process(j) 
    }
}
```
---

## 4. Backpressure & Reliability
Channel Saturation: If the buffered channel is full, the system returns a 429 Too Many Requests (Backpressure mechanism).

Fault Tolerance: If Kafka is temporarily unreachable, the workers implement a retry logic with exponential backoff before dropping the packet.