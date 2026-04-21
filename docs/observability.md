# Observability & Monitoring Strategy

This document outlines a multi-tiered monitoring system designed for high-intensity IoT data (5-second intervals, 2000+ RPS).

## 1. SLIs & SLOs: Performance, Errors & Saturation

### A. Throughput & Latency
| Metric (SLI) | Definition | SLO (Target) |
| :--- | :--- | :--- |
| **Ingress RPS** | Intensity of incoming packets in Go Ingestor. | 2,000 (Nominal) / 5,000 (Peak) |
| **Response Time** | Processing time for telemetry packets. | **< 100ms** (Success) |
| **Bandwidth Usage** | Network interface load on Ingress layer. | **< 80%** Capacity |

### B. Error Rates & Reliability
| Metric (SLI) | Definition | SLO (Target) |
| :--- | :--- | :--- |
| **Request Errors** | Rate of 5xx/4xx HTTP/MQTT errors. | **< 0.1%** of total traffic |
| **DB Lookup Rate** | Go requests falling back to PostgreSQL due to Redis cache miss. | **< 5%** (Critical for scaling) |
| **Hardware Errors** | Disk I/O errors or packet loss at the NIC level. | **0 errors/min** |

### C. Resource Saturation
| Metric (SLI) | Component | Threshold / SLO |
| :--- | :--- | :--- |
| **Pool Saturation** | DB Connection Pool usage (TimescaleDB). | **< 80%** saturation |
| **Cache Capacity** | Redis memory utilization (Key-value store). | **< 90%** (Prevents fallback to DB) |

## 2. The Role of Redis in Observability
Redis acts as the system's "Operational Memory." Alerting on **CacheExhaustion (>90%)** is critical because Redis stores:
1. **Device Binding:** Mapping `DeviceID` to `PetID` & `OwnerID`.
2. **Dynamic Thresholds:** Breed-specific heart rate limits for instant anomaly detection.
3. **Heartbeat/Status:** Last seen timestamp and battery level for rapid UI updates.

## 3. Alerting Classification (By Severity & Level)

### P0: Critical (Immediate Response)
* **ServiceDown:** Go or Java service is unreachable.
* **DatabaseFull:** TimescaleDB storage exceeds 85%.
* **Stale Device (Business Alert):** No communication from a registered device for **> 48 hours**.

### P1: High Priority (Technical/Safety)
* **Device Brute-force:** > 5 authentication errors for a specific Device ID.
* **Sensor Failure:** Device reports hardware error (Pulse 0 logic check).
* **Thermal Shutdown:** Device internal temp > 65°C (Safety risk for the pet).

### P2: Medium (Maintenance)
* **High Latency:** P95 response time > 200ms.
* **Battery Low:** Device battery level < 10%.
* **Clock Drift:** Time difference between device and server > 10s.

## 4. Multi-Level Connectivity Alerts (Silence Grading)
To avoid "alert fatigue," we use graded response times for device silence:
* **1 min:** Info (Potential Wi-Fi jitter) -> Logs only.
* **15 min:** Low (Device Offline) -> Push notification to Owner.
* **2 hours:** Medium (Critical Silence) -> CRM Alert for the Clinic.
* **48 hours:** High (Stale Device) -> Direct contact by Clinic Manager.

## 5. Tooling Stack
* **Prometheus & Grafana:** Infrastructure and technical metrics (CPU, RPS, Latency).
* **Micrometer (Java):** JVM health, GC pauses, and API performance.
* **Prometheus Go Client:** Custom metrics for Worker Pool saturation and Batch sizes.
* **ELK Stack (Elasticsearch, Logstash, Kibana):** Tracking device degradation trends (e.g., "Which collar models overheat most often?").
* **SQL Exporter:** Monitoring TimescaleDB compression efficiency and hypertable sizes.