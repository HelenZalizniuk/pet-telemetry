# 🏗️ Architectural Requirements & Project Concept

## 1. Project Vision & Strategic Evolution
The system is evolving from a legacy monolithic Java application (PetClinic) into a **high-load IoT monitoring ecosystem**.
* **Core Objective:** Remote monitoring of pet vitals (heart rate, temperature, etc.) via smart "vitals collars."
* **Technological Pivot:** Implementing a **Go-based Ingestor** to handle high-throughput streams (2,000+ RPS) while maintaining the stable Java core for complex business logic and UI.

## 2. Architectural Requirements Assessment
Requirements are evaluated based on their **Architectural Significance Score (0-10)**, reflecting their impact on the system design.

### A. Functional Requirements (FR)
| ID | Requirement | Description | Significance |
| :--- | :--- | :--- | :--- |
| **FR-1** | **Telemetry Ingestion** | Support for high-frequency data streams (5s interval) via **MQTT/REST**. | **10** |
| **FR-2** | **Device Management** | Flexible binding/unbinding of collars to different pets (Reusability). | **7** |
| **FR-3** | **Real-time Alerting** | Instant threshold validation using in-memory logic (Redis) to avoid DB bottlenecks. | **8** |
| **FR-4** | **CRM Integration** | Synchronization of telemetry data with existing medical records in the Java module. | **7** |

### B. Non-Functional Requirements (NFR)
| ID | Requirement | Description | Significance |
| :--- | :--- | :--- | :--- |
| **NFR-1** | **Scalability** | Capability to process data from 10,000+ concurrent devices (2,000+ RPS). | **10** |
| **NFR-2** | **Storage Efficiency** | Utilizing 90%+ data compression in TimescaleDB for long-term historical storage. | **9** |
| **NFR-3** | **Availability** | 99.9% uptime for the critical ingestion path (High-availability for health monitoring). | **9** |
| **NFR-4** | **Low Latency** | End-to-end latency from data ingestion to alert triggering < 1 second. | **8** |

### C. Constraints (CON)
| ID | Requirement | Description | Significance |
| :--- | :--- | :--- | :--- |
| **CON-1** | **Hybrid Tech Stack** | New high-load services: **Golang**; Existing CRM: **Java/Spring**. | **10** |
| **CON-2** | **Unified Database** | Standardizing on **TimescaleDB** (PostgreSQL) for both relational and time-series data. | **9** |
| **CON-3** | **Power Efficiency** | Optimized protocols for IoT devices to save battery (Preferring MQTT over HTTP). | **7** |

---

## 3. Stakeholder Questionnaire (The Architect's Focus)
*Crucial questions identified to define the system's architectural boundaries:*

1.  **Connection Loss Behavior:** Should the collar buffer data locally (e.g., for 10-20 min) and upload it in bursts, or is data loss during downtime acceptable?
2.  **Adaptive Sampling:** Can the 5-second interval be adjusted (e.g., to 1 min when the pet is sleeping) to extend battery life?
3.  **Data Retention & Aging:** How long is "per-second" precision required (e.g., 7 days) before data is aggregated into 1-minute averages for long-term storage?
4.  **Maintenance Lifecycle:** Is a dedicated "Decontamination/Service" status needed for devices between assignments to different pets?
5.  **Peak Ingress Scenarios:** What is the anticipated "thundering herd" profile (e.g., all devices reconnecting simultaneously after a network outage)?

---

## 4. Protocol Rationale (MQTT vs. HTTP)
For the Go-based Ingress layer, **MQTT** was prioritized for the following reasons:
* **Minimal Overhead:** Extremely small packet headers to save pet collar battery and network bandwidth.
* **Concurrency:** Go's goroutines efficiently manage tens of thousands of persistent MQTT connections.
* **Reliability:** Built-in Quality of Service (QoS) levels ensure critical health alerts are delivered.

## 5. The Role of Redis in Scaling
Redis serves as the "Operational Memory" for the ingestion layer, handling:
* **Device Routing Table:** Instant `DeviceID` -> `PetID` mapping.
* **Health Thresholds:** Storing breed-specific heart rate/temp limits for zero-latency anomaly detection.
* **Last Seen Status:** Rapid "Online/Offline" status updates without querying the main SQL database.