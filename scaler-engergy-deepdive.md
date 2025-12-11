# Scalar Platform – Deep Dive (Architecture, Data Flow & Operations)

## 📌 Document Purpose
This document explains the Scalar energy data platform in a **comprehensive, high-detail** manner.
It focuses on:

- System architecture & major components
- Data acquisition → ingestion → analytics end-to-end flow
- Docker Compose architecture
- Observability & monitoring
- Message broker (Pulsar) and StarRocks usage
- Dependencies between services
- How the platform should map into Azure for Phase-1 modernization
- Operational considerations

---

# 1. 🌐 High-Level System Overview

Scalar is a **microservices-based**, containerized data platform used to collect, process, and analyze real-time and historical European energy market data.

## Core Responsibilities
- **Data Acquisition:** Pull data from Elia, Entsoe, Nordpool, Energinet, Meteologica, Volue, JAO
- **Data Ingestion:** Validate, transform, and persist data
- **Analytics & Forecasting:** Python-based ML models
- **Messaging Backbone:** Apache Pulsar (broker + zookeeper + bookkeeper)
- **Analytics Storage:** StarRocks FE/BE
- **Realtime Cache:** Redis Stack
- **Observability:** OpenTelemetry + LGTM stack (Grafana + Loki + Tempo + Mimir)

---

# 2. 🧩 Major Components

## 2.1 Acquisition Service (.NET 9 Workers)
**Purpose:** Fetch external market data → publish raw data to Pulsar.

### Responsibilities
- Manage scheduled workers
- Call external provider APIs
- Handle retry logic (Polly)
- Emit telemetry
- Publish structured messages to Pulsar topics

### Worker Examples
- Elia.AcquisitionWorker (15+)
- JAO.AcquisitionWorker (6)
- Nordpool realtime
- Energinet data
- Meteologica forecasting data

### Ports
- **7001 (exposed)**
- **8080 (internal)**

---

## 2.2 Ingestion Service (.NET 9)
**Purpose:** Consume Pulsar messages → validate → transform → batch insert into StarRocks → update Redis.

### Responsibilities
- Subscribe to Pulsar topics
- Validate provider-specific payloads
- Transform to internal models
- Perform **batch inserts** into StarRocks
- Update Redis with time-sensitive fields
- Emit metrics/traces

### Batch Configuration
- MaxBatchSize = 100
- MaxBatchWaitTimeSeconds = 30
- Compaction every 60 min

### Ports
- **7002 (exposed)**
- **8080 (internal)**

---

## 2.3 Pulsar Messaging Layer
Scalar uses **Apache Pulsar 4.0.5** with:

- **Zookeeper** for metadata/coordination
- **BookKeeper Bookie** for distributed persistence
- **Pulsar Broker** for producer/consumer communication

### Why Pulsar?
- High throughput
- Low latency
- Partitioning support
- Strong durability with BookKeeper ledgers
- Built-in multi-tenancy and geo-replication support

---

## 2.4 StarRocks OLAP Engine (3.4)
StarRocks consists of:

### Frontend (FE)
- SQL parser & optimizer
- Metadata storage
- Query planner

### Backend (BE)
- Storage engine
- Query executor

### Usage
- Time-series analytics
- Forecasting model input
- Dashboard backend

---

## 2.5 Redis Stack (7.4)
Used for:
- Real-time state
- Caching
- Fast in-memory lookups
- Temporary analytics snapshots

---

## 2.6 Python Insights Service
Used for ML forecasting and day-ahead strategy.

### ML Models
- Nordic imbalance
- Baltics imbalance
- DK demand forecasting
- DK day-ahead strategy
- AutoGluon-based pipelines

### Responsibilities
- Run scheduled forecast tasks
- Retrieve data from StarRocks
- Publish forecast results to Pulsar

---

## 2.7 LGTM Observability Stack
Single container for:

- **Grafana** – dashboards
- **Loki** – logs
- **Tempo** – traces
- **Mimir** – metrics

---

# 3. 🔁 End-to-End Data Flows

## 3.1 Acquisition Flow
External Provider → Acquisition Worker → Pulsar Topic

**Steps**
1. Worker triggers on schedule
2. Fetches data via HTTP/SFTP/API
3. Parses, validates basic schema
4. Sends message to Pulsar topic
5. Logs tracing & metrics

---

## 3.2 Ingestion Flow
Pulsar Topic → Ingestion Worker → StarRocks (batch insert)
                                     → Redis Cache (realtime)

**Steps**
1. Subscribes to Pulsar
2. Reads messages
3. Validates payload
4. Transforms to internal DTO
5. Persists to StarRocks
6. Updates Redis cache

---

## 3.3 Insights (ML) Flow
StarRocks → Python ML Service → Pulsar (forecast events)

**Steps**
1. Worker retrieves historical dataset
2. Runs trained AutoGluon model
3. Generates forecast
4. Publishes to Pulsar topic
5. Downstream services read predictions

---

# 4. 🧱 Docker Compose Architecture (Current Implementation)

```
starrocks-fe
starrocks-be
zookeeper
pulsar-init
bookie
broker
redis
acquisition
ingestion
otel-lgtm
otel-collector
```

### Key Notes
- FE must start before BE
- BookKeeper requires Zookeeper
- Broker requires Zookeeper + BookKeeper
- acquisition/ingestion depend on Redis + Pulsar
- LGTM collects telemetry from all services

---

# 5. 🔐 Secret Management (Current vs Azure)

## Current State (Local/Docker)
Secrets are provided via:
- Environment variables
- Key Vault integration for .NET workers

Examples:
- EntsoeApi--SecurityToken
- Nordpool credentials
- Pulsar connection
- Redis host
- StarRocks host + credentials
- OTLP endpoint
- Azure TenantID, ClientID, Secret

## Azure Best Practice
Store secrets in Azure Key Vault:

| Secret Category | Example |
|----------------|---------|
| Azure Identity | scalar-tenant-id, client-id, client-secret |
| API Tokens | elia, entsoe, nordpool, jao |
| Redis | redis-host, redis-port |
| StarRocks | db-host, db-user, db-pass |
| Pulsar | broker-url, service-url |
| Observability | otlp-endpoint |

---

# 6. 🔎 Observability & Telemetry

All services emit telemetry through OpenTelemetry SDK.

### Flow
Service → OTLP Exporter → otel-collector → LGTM Stack → Grafana Dashboard

### What is captured?
- Service logs
- Request-level traces
- Internal worker metrics
- Pulsar consumer lag
- Batch insert performance

---

# 7. 🏗️ Azure Mapping – Phase 1 Modernization

## 7.1 Compute Mapping
| Component | Azure Equivalent |
|----------|------------------|
| acquisition | Azure App Service (Linux + Docker) |
| ingestion | Azure App Service |
| insights (python) | App Service or Container Apps |
| redis | Azure Cache for Redis |
| starrocks | Azure VM Scale Set |
| pulsar | Azure VMs |
| otel-lgtm | VM or Managed Grafana |
| otel-collector | VM or Container Apps |

---

## 7.2 Networking
### VNet Layout

vnet-scalar
│
├── subnet-appservice
├── subnet-data
└── subnet-monitoring

### NSG Best Practice
Use **subnet-level NSG only**.

---

## 7.3 Private Endpoints
Recommended for:
- Redis
- Key Vault
- Storage (ML models)

---

# 8. ⚙️ Operational Considerations

### Startup Order
1. starrocks-fe
2. starrocks-be
3. zookeeper
4. pulsar-init
5. bookie
6. broker
7. redis
8. acquisition
9. ingestion
10. insights
11. otel-lgtm
12. otel-collector

### Failure Scenarios
- Pulsar failure → acquisition/ingestion freeze  
- Redis failure → ingestion cache disabled  
- StarRocks failure → ingestion stops  

---

# 9. 📈 Scalability Overview

## Horizontal Scaling
- Replicate ingestion workers
- Add Pulsar partitions
- Add bookies
- Redis clustering

## Vertical Scaling
- Increase FE/BE compute
- Bigger VMs for Pulsar cluster

---

# 10. 🔮 Future Phase (AKS Migration – Summary)

- Kubernetes for acquisition/ingestion/insights
- Pulsar Operator or StreamNative
- StarRocks Operator
- Redis Enterprise or Azure Cache
- Managed Grafana

---

# 11. 📦 Ports Summary

| Service | Ports |
|--------|-------|
| starrocks-fe | 8030, 9020, 9030 |
| starrocks-be | 8040 |
| redis | 6379 |
| broker | 6650, 8080 |
| zookeeper | 2181 |
| acquisition | 7001 |
| ingestion | 7002 |
| otel-lgtm | 3000, 4317, 4318 |
| otel-collector | 4317 |

---

# 12. 🧭 Project Structure (Simplified)

scalar-mono/
  Acquisition/
  Ingestion/
  Signals/
  Execution/
  Interface/
  Insights/
  Shared/
  Scalar.DAL/
  docker-compose.yaml
  Scalar.sln

---

# 13. 📝 Daily Operations Checklist

### StarRocks
- FE metadata health
- BE storage usage
- Query latency

### Pulsar
- Consumer backlog
- BookKeeper disk usage
- Zookeeper quorum

### Ingestion
- Batch insert times
- Failed messages
- Redelivery count

### Acquisition
- API throttling
- Provider errors
- Worker uptime

### Insights (ML)
- Forecast completion
- Pulsar publication

### Observability
- Grafana dashboards
- Loki log trends
- Tempo trace traffic
- Mimir retention

---

# 🎉 End of Document
