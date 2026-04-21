## 🎯 Architectural Requirements & Rationale

I evaluated the system requirements based on their **Architectural Significance Score (0-10)**. This score reflects how much the requirement shapes the final system design.

### 1. Functional Requirements (FR)
| ID | Requirement | Description | Significance (0-10) |
| :--- | :--- | :--- | :--- |
| **FR-1** | **High-Load Ingestion** | Support for 2,000+ RPS from IoT collars. | **10** |
| **FR-2** | **Real-time Alerting** | Immediate notification delivery for health anomalies. | **8** |
| **FR-3** | **Historical Analysis** | Storage and retrieval of telemetry for vets (last 30 days). | **6** |
| **FR-4** | **Data Integration** | Synchronization with existing Java-based PetClinic CRM. | **7** |

### 2. Non-Functional Requirements (NFR)
| ID | Requirement | Description | Significance (0-10) |
| :--- | :--- | :--- | :--- |
| **NFR-1** | **Availability** | 99.9% uptime (Critical for health monitoring). | **9** |
| **NFR-2** | **Scalability** | Horizontal scaling to 100k+ devices without redesign. | **10** |
| **NFR-3** | **Reliability** | Zero data loss for processed health alerts (Outbox pattern). | **9** |
| **NFR-4** | **Latency** | Ingestion P95 latency < 100ms. | **8** |

### 3. Constraints (CON)
| ID | Requirement | Description | Significance (0-10) |
| :--- | :--- | :--- | :--- |
| **CON-1** | **Legacy Sync** | Must coexist with the legacy Java monolithic database. | **8** |
| **CON-2** | **Cloud-Native** | Must be deployable in Kubernetes environments. | **5** |
| **CON-3** | **Protobuf/MQTT** | Specific protocols mandated for IoT device communication. | **6** |

## ❓ Stakeholder Questionnaire
*Crucial questions asked to define the architectural boundaries:*

1. **Traffic Spikes:** What is the expected peak-to-average ratio? (e.g., Do all collars send data at the exact same second?)
2. **Data Retention:** What is the legal/business requirement for keeping raw telemetry? (Determines TimescaleDB compression policies).
3. **Consistency vs Availability:** In case of a regional outage, is it more critical to keep receiving heartbeats or to ensure the vet sees the absolute latest record? (PACELC validation).
4. **IoT Device Capabilities:** Do the collars support local buffering/retries if the ingestion endpoint is unreachable?
5. **Security:** Are there specific compliance requirements (like GDPR for pet owners' data) that impact data sharding?