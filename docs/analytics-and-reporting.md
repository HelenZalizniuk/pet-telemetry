# Analytics Engine: Data Flow & Reporting Design

This document describes the design of the analytics subsystem, detailing how raw telemetry is transformed into actionable medical insights for pet owners and veterinarians.

## 1. System Overview
The analytics engine processes data from two main streams:
1.  **Vitals Stream (IoT):** Real-time heart rate, temperature, and activity levels.
2.  **Clinical Metadata (Java CRM):** Pet profiles, breed standards, and historical medical records.

## 2. Reporting Strategy & Processing Models

I have categorized the required reports based on their processing requirements (Streaming vs. Batching).

### Report A: Real-time Health Status (Current Vitals)
* **Goal:** Total current value of health indicators for a pet at any given moment.
* **Processing:** **Streaming**
* **Design:** Data is ingested via Go and pushed to a **Redis-based Live Dashboard**. We use streaming because immediate detection of a "current" state is critical for emergency responses.
* **Logic:** Go Ingestor -> Kafka -> Real-time Processor -> Redis (Current State) -> WebSocket to UI.

### Report B: Historical Trends (Daily/Monthly/Yearly changes)
* **Goal:** Changes in pet vitals over a selected period (numbers and percentages).
* **Processing:** **Batching (via Continuous Aggregates)**
* **Design:** Since pets generate millions of data points, recalculating them on every request is inefficient. We use **TimescaleDB Continuous Aggregates**.
* **Logic:** Raw data is downsampled into 1-hour buckets every night. The Java API queries these pre-calculated buckets, providing a 90% speed boost for long-term reports.

### Report C: Comparative Market/Breed Analysis
* **Goal:** Comparing an individual pet's performance against "Market Benchmarks" (average vitals for the breed/age group) to identify top/bottom health performers.
* **Processing:** **Hybrid (Lambda Architecture)**
* **Design:** We use **Batching** for breed averages (calculated once a day) and **Streaming** for the individual pet's current data.
* **Logic:** A Spark/Flink job calculates breed-wide averages (The "Market Trend"). The reporting service joins this "cold" data with the pet's "hot" data to highlight deviations.

## 3. High-Level Data Flow Diagram

1.  **Ingestion:** IoT Collars -> MQTT -> **Go Ingestor** (Validation).
2.  **Transport:** Go Ingestor -> **Apache Kafka** (Buffering).
3.  **Processing Layer:**
    * *Path 1 (Speed):* Kafka -> Flink/Go-Worker -> Redis (Alerts & Live View).
    * *Path 2 (Batch):* Kafka -> **TimescaleDB** (Hypertable Storage).
4.  **Presentation:** **Java Medical Service** (API) -> BFF -> Mobile/Web App.

## 4. Tech Stack & Storage Selection

| Component | Recommendation | Rationale |
| :--- | :--- | :--- |
| **Stream Processing** | **Kafka Streams / Go Workers** | High-speed processing of 2,000+ RPS with low latency. |
| **Batch Processing** | **TimescaleDB Continuous Aggs** | Eliminates the need for a separate Hadoop cluster by using DB-native aggregation. |
| **Cold Storage** | **S3 / TimescaleDB** | Archiving raw data for legal/compliance needs. |
| **Analytics DB** | **TimescaleDB (PostgreSQL)** | SQL-compatibility allows vets to run complex analytical queries (Breed vs. Health). |
| **Visualization** | **Grafana / Custom React UI** | Real-time charts and historical trend visualization. |

## 5. Justification of Choices
* **Why Streaming for Report A?** In health monitoring, "Current State" must be sub-second. Batching would introduce unacceptable delays.
* **Why Batching for Report B?** Historical data is immutable. Pre-calculating it saves CPU and improves the User Experience by making dashboards "snappy."
* **Why Hybrid for Report C?** Breed benchmarks don't change every second. Calculating them once a day (Batch) and comparing them to real-time data (Streaming) is the most cost-effective approach.