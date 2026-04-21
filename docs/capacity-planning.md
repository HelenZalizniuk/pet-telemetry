# Capacity Planning & Infrastructure Analysis

This document outlines the resource requirements and throughput calculations for the **pet-telemetry** system.

## 1. Throughput & Traffic Estimates
Based on the target of 2,000 requests per second (RPS) from 10,000+ active devices:

| Metric | Calculation | Result |
| :--- | :--- | :--- |
| **Incoming RPS** | Target benchmark | 2,000 req/sec |
| **Payload Size (Protobuf)** | Average message | ~150 Bytes |
| **Ingress Bandwidth** | 2,000 * 150 bytes | **~300 KB/s** |
| **Daily Message Count** | 2,000 * 86,400 sec | **172.8 Million msgs/day** |
| **Raw Data per Day** | 172.8M * 150 bytes | **~25.9 GB/day** |

## 2. Storage Strategy (TimescaleDB)
To manage costs while maintaining performance, we apply a multi-tier retention policy:

* **Hot Storage (First 7 days):** Uncompressed, high-speed access for real-time monitoring. (~180 GB)
* **Compressed Storage (7-30 days):** Utilizing TimescaleDB native compression (up to 90% reduction).
* **Total Monthly Footprint:** Estimated **~350-400 GB** after compression and indexing overhead.

## 3. Compute Resources (Go Ingestor)
To handle 2,000 RPS with a sub-100ms P95 latency, the Go Ingestor is provisioned as follows:

* **Concurrency Model:** Worker Pool with 50-100 workers per instance.
* **Instance Scaling:** * Minimum: 3 pods (for High Availability).
    * Maximum: 10 pods (during peak spikes).
* **Average Latency Target:** 40-60ms per request.

## 4. Infrastructure Cost Projection (Monthly TCO)
*Estimation based on AWS/Cloud managed services:*

| Service | Component | Estimated Cost |
| :--- | :--- | :--- |
| **Compute** | AWS EKS (t3.medium nodes) | $120 |
| **Database** | Managed TimescaleDB/PostgreSQL | $100 |
| **Broker** | Managed Kafka (small cluster) | $120 |
| **Cache** | Redis (t3.micro) | $20 |
| **Total** | | **~$360 / month** |