# Scalar Energy Platform - Architecture Documentation

## Table of Contents
- [System Overview](#system-overview)
- [Architecture Principles](#architecture-principles)
- [Technology Stack](#technology-stack)
- [System Components](#system-components)
- [Data Flow Architecture](#data-flow-architecture)
- [Infrastructure & Deployment](#infrastructure--deployment)
- [Security & Secrets Management](#security--secrets-management)
- [Monitoring & Observability](#monitoring--observability)
- [Development Guidelines](#development-guidelines)

---

## System Overview

Scalar is a **microservices-based energy data platform** built on .NET 9.0 that acquires, processes, ingests, and analyzes energy market data from multiple European energy providers and exchanges. The platform is designed to handle real-time data ingestion, processing, and analytics for energy trading and market analysis.

### Core Purpose
- **Data Acquisition**: Collect energy market data from multiple providers (Elia, Entsoe, Nordpool, JAO, Energinet, Meteologica, Volue)
- **Data Ingestion**: Process and persist collected data into analytical data stores
- **Signal Processing & Forecasting**: Generate trading signals, analytics, and ML-based forecasts from ingested data
- **Execution**: Execute trading strategies based on processed signals
- **User Interface**: Provide visualization and interaction capabilities

---

## Architecture Principles

1. **Microservices Architecture**: Independent, loosely coupled services with single responsibilities
2. **Event-Driven Communication**: Apache Pulsar message broker for asynchronous service communication
3. **Separation of Concerns**: Clear boundaries between acquisition, ingestion, processing, and presentation
4. **Shared Libraries**: Common functionality centralized in the Shared project
5. **Containerization**: Docker-based deployment for consistency and portability
6. **Observability First**: Built-in monitoring, metrics, and distributed tracing

---

## Technology Stack

### Backend Framework
- **.NET 9.0**: Core runtime and framework
- **C# 9.0+**: Primary programming language
- **ASP.NET Core**: Web application framework

### Data Storage
- **StarRocks 3.4**: OLAP database for analytical workloads (distributed query engine)
  - Frontend (FE): Query planning and metadata management
  - Backend (BE): Data storage and query execution
- **Redis Stack**: In-memory data store for caching and real-time data
- **MySQL**: Relational database via MySqlConnector

### Messaging & Communication
- **Apache Pulsar 4.0.5**: Message broker for event streaming and service communication
- **DotPulsar (v4.3.1)**: Native .NET client for Apache Pulsar
- **Server-Sent Events (SSE)**: Real-time data streaming to clients

### Monitoring & Observability
- **Grafana OpenTelemetry (v1.2.0)**: Metrics, logs, and traces collection
- **LGTM Stack** (Loki, Grafana, Tempo, Mimir): Complete observability platform
- **OpenTelemetry Protocol (OTLP)**: Telemetry data export

### Security & Configuration
- **Azure Key Vault**: Secrets management via Azure.Extensions.AspNetCore.Configuration.Secrets
- **Azure Identity (v1.15.0)**: Authentication with Azure services
- **Azure AD**: Service principal authentication

### Data Sources & APIs
- **Elia**: Belgian transmission system operator (Scalar.Elia.Integration) - Primary provider with 15+ workers
- **Entsoe**: European Network of Transmission System Operators (Scalar.EntsoeClient + Slinky.Entsoe v0.0.8)
- **JAO**: Joint Allocation Office (Scalar.JAO.Client)
- **Nordpool**: Nordic power exchange (NPS.ID.PublicApi)
- **Energinet**: Danish energy data service
- **Meteologica**: Weather and renewable energy forecasting
- **Volue**: Energy market data and analytics
- **Baltic Transparency**: Baltic energy market data

### Frontend (Interface)
- **Blazor Server**: Interactive web UI with server-side rendering
- **Radzen Components**: UI component library for Blazor
- **Razor Components**: Component-based UI framework

### Infrastructure
- **Docker**: Container runtime
- **Docker Compose**: Multi-container orchestration
- **SSH.NET (v2025.0.0)**: Secure file transfer for data acquisition

### Key Libraries
- **Polly (v8.6.3)**: Resilience and transient fault handling
  - Core retry/circuit breaker policies
  - Rate limiting capabilities
- **Redis.OM (v1.0.1)**: Object mapping for Redis
- **Newtonsoft.Json (v13.0.3)**: JSON serialization

---

## System Components

### 1. Acquisition Service
**Purpose**: Fetches raw energy market data from external providers

**Port**: 7001 (Docker), 8080 (Internal)

**Key Responsibilities**:
- Connect to external data provider APIs
- Schedule and execute data retrieval jobs
- Publish raw data to message broker
- Handle API rate limiting and retries

**Workers** (40+ total, environment-specific):
- **Development Environment** (Active):
  - `Elia.AcquisitionWorker` (15+ workers - all modules)
  - `JAO.AcquisitionWorker` (6 workers)
  - `Nordpool.Realtime` services

- **Production Environment** (Active):
  - All Development workers plus:
  - `Energinet.AcquisitionWorker`
  - `Entsoe.CurrentBalancingStateWorker` (fallback mode)
  - `Nordpool.Rest.MarketData.MarketDataAcquisition`
  - `Meteologica.AcquisitionWorker`
  - `Volue.AcquisitionWorker`

**Configuration**:
- Workers can be enabled/disabled via database configuration seeding
- Separate authentication tokens per provider/API endpoint
- Environment-specific URLs (test vs production)

**Dependencies**:
- Shared library (clients, models, monitoring)
- External API clients (Elia, Entsoe, JAO, Nordpool, etc.)
- Apache Pulsar (message publishing)
- Redis (caching/state management)

**Technology**:
- ASP.NET Core Worker Service
- Hosted background services
- HTTP clients with resilience policies (Polly)

---

### 2. Ingestion Service
**Purpose**: Consumes raw data from message broker and persists to data stores

**Port**: 7002 (Docker), 8080 (Internal)

**Key Responsibilities**:
- Subscribe to data acquisition messages
- Transform and validate incoming data
- Persist data to StarRocks/databases
- Ensure data quality and consistency
- Batch processing for optimized database writes

**Workers** (19+ total, environment-specific):
- **Development Environment** (Active):
  - `Elia.IngestionServices` (all modules)
  - `JAO.IngestionServices`
  - `Nordpool.Intraday.Realtime`

- **Production Environment** (Active):
  - All Development workers plus:
  - `Energinet.IngestionWorker`
  - `Entsoe.IngestionWorker`
  - `Nordpool.Rest.MarketDataIngestion`
  - `Meteologica.IngestionWorker`
  - `Volue.IngestionWorker`
  - `Scalar.Forecast.Services`

**Batch Processing Configuration**:
- MaxBatchSize: 100 records
- MaxBatchWaitTimeSeconds: 30
- CompactionThreshold: 5000
- CompactionIntervalMinutes: 60

**Dependencies**:
- Shared library (persistence, models, monitoring)
- Apache Pulsar (message consumption)
- StarRocks database
- Redis

**Data Flow**:
```
Pulsar Topic → Consumer → Data Validation → Transformation → Batch Insert → StarRocks
                                                           → Cache Update → Redis
```

---

### 3. Signals Service
**Purpose**: Analyzes ingested data to generate trading signals and insights

**Port**: 7004 (Docker), 8080 (Internal) - Currently commented out

**Key Responsibilities**:
- Process historical and real-time energy data
- Execute signal generation algorithms
- Publish signals to downstream services
- Perform statistical analysis

**Status**: Partially implemented, currently disabled in docker-compose

**Dependencies**:
- Shared library
- StarRocks (data queries)
- Apache Pulsar (signal publishing)

---

### 4. Execution Service
**Purpose**: Executes trading strategies based on generated signals

**Port**: 7003 (Docker), 8080 (Internal) - Currently commented out

**Key Responsibilities**:
- Consume trading signals
- Execute trading logic
- Manage positions and orders
- Risk management

**Status**: Framework in place, currently disabled in docker-compose

**Dependencies**:
- Shared library
- Apache Pulsar (signal consumption)
- StarRocks (historical data)

---

### 5. Interface Service
**Purpose**: Web-based user interface for visualization and interaction

**Port**: 8888 (Docker), 8080 (Internal) - Currently not deployed

**Key Responsibilities**:
- Display energy market data
- Visualize trading signals
- Provide analytics dashboards
- User interaction and configuration

**Technology**:
- Blazor Server (interactive server-side rendering)
- Radzen UI components (v7.0.7)
- Razor components

**Features** (Planned):
- Real-time data updates
- Interactive charts and graphs
- Signal monitoring
- System health monitoring

**Status**: Framework ready, minimal implementation, not currently deployed

---

### 6. Insights Service (Python ML Service)
**Purpose**: Machine learning-based forecasting and analytics service

**Port**: 8000 (Docker)

**Key Responsibilities**:
- Generate ML-based forecasts (imbalance, demand, price)
- Publish predictions to Pulsar topics
- Execute day-ahead and intraday trading strategies
- Provide model backtesting capabilities

**Technology Stack**:
- Python 3.x runtime
- AutoGluon for ML forecasting
- FastAPI for web interface
- Pulsar Python client for message publishing
- StarRocks client for data retrieval

**Key Models**:
- **Nordic Imbalance Forecast**: ACRDLS (Autocorrelation RDLS) model
- **Baltics Imbalance**: Regional imbalance predictions
- **DK Balancing Demand**: Danish market demand forecasting
- **DK Day-Ahead Strategy**: Day-ahead market strategy generator

**Integration**:
- Separate docker-compose deployment
- Connects to main scalar-network
- OpenTelemetry integration (OTLP endpoint)
- Azure Blob Storage for model persistence
- Direct Pulsar publisher for forecast distribution

**Deployment**: Standalone container connected to main infrastructure network

---

### 7. Shared Library
**Purpose**: Common functionality shared across all .NET services

**Key Modules**:

#### Clients (`Shared/Clients/`)
External API client implementations:
- `Elia`: Belgian TSO client (custom Scalar.Elia.Integration)
- `Entsoe`: ENTSO-E transparency platform client (custom Scalar.EntsoeClient)
- `JAO`: Joint Allocation Office client (custom Scalar.JAO.Client)
- `Energinet`: Danish energy data service client
- `Nordpool`: Nordic power exchange client
- `Meteologica`: Weather/forecast data client
- `Volue`: Energy analytics API client
- `Common`: Shared client utilities

#### Persistence (`Shared/Persistence/`)
Database and storage abstractions:
- `Starrocks`: OLAP database operations
- `SQL`: SQL database helpers (via Scalar.DAL)
- `Memory`: In-memory caching (Redis)
- Provider-specific transactions: `Elia`, `Entsoe`, `JAO`, `Energinet`, `Nordpool`, `Meteologica`, `Volue`
- `Common`: Shared persistence utilities

#### Communication (`Shared/Communication/`)
Message broker abstractions:
- `Pulsar.cs`: Apache Pulsar client configuration
- `Producing`: Message producer utilities
- `Consuming`: Message consumer utilities
- `Reading`: Message reading patterns
- `Extensions`: Helper extensions
- `EnumMaps.cs`: Message type mappings

#### Models (`Shared/Models/`)
Data models and DTOs:
- Provider-specific models (Elia, Entsoe, JAO, Energinet, Nordpool, Meteologica, Volue)
- Response/Request DTOs
- Persistence models
- Domain entities

#### Monitoring (`Shared/Monitoring/`)
Observability infrastructure:
- OpenTelemetry integration
- Metrics collection
- Distributed tracing
- Logging configuration

#### Secrets (`Shared/Secrets/`)
Secrets management:
- Azure Key Vault integration
- Configuration providers
- Credential management

---

## Data Flow Architecture

### 1. Acquisition Flow
```
External API → Acquisition Worker → Pulsar Topic → Ingestion Worker → StarRocks
                                                                     → Redis Cache
```

**Steps**:
1. Scheduled worker triggers data fetch from external provider
2. Worker retrieves data via HTTP/SFTP/API
3. Raw data published to provider-specific Pulsar topic
4. Acknowledgment received from broker

### 2. Ingestion Flow
```
Pulsar Topic → Consumer → Validation → Transformation → Batch Insert → StarRocks
                                                      → Cache Update → Redis
```

**Steps**:
1. Ingestion worker subscribes to Pulsar topic
2. Receives and deserializes messages
3. Validates data integrity and schema
4. Transforms to internal data model
5. Batch inserts into StarRocks for analytics
6. Updates Redis cache for real-time access

### 3. Signal Generation Flow (Planned)
```
StarRocks Query → Signal Algorithm → Signal Event → Execution Service
                                   → Pulsar Topic → Interface Service
```

### 4. Execution Flow (Planned)
```
Signal Event → Strategy Evaluation → Order Generation → External Exchange
            → Risk Check           → Position Update → StarRocks
```

---

## Infrastructure & Deployment

### Docker Compose Architecture

**Network**: `scalar-network` (bridge driver)

**Service Dependencies**:
```
starrocks-fe (base)
    ↓
starrocks-be
    ↓
zookeeper (Pulsar coordination)
    ↓
pulsar-init (cluster metadata)
    ↓
bookie (ledger persistence)
    ↓
broker (Pulsar message broker)
    ↓
redis
    ↓
acquisition, ingestion, [insights (separate compose)]
    ↓
otel-lgtm (observability)
```

**Note**: Execution, Signals, and Interface services have framework structure but are not deployed

### Volume Management
- `data_starrocks_fe_meta`: StarRocks frontend metadata
- `data_starrocks_be_storage`: StarRocks backend data storage
- `pulsar_data_zookeeper`: Zookeeper coordination state
- `pulsar_data_bookkeeper`: BookKeeper ledger storage
- `data_redis`: Redis data persistence
- `data_monitoring`: Grafana/LGTM observability data

### Health Checks
- **StarRocks FE**: TCP connection to port 9030
- **StarRocks BE**: TCP connection to port 8040
- **Pulsar Broker**: Admin API health check (`/admin/v2/brokers/health`)
- **Zookeeper**: Ruok command
- **BookKeeper**: Bookie sanity check
- **Redis**: PING command

### Environment Configuration
Each service uses:
- `ASPNETCORE_ENVIRONMENT`: Development/Production
- `DOTNET_ENVIRONMENT`: Development/Production
- `APP_RUNNING_LOCALLY`: false (in Docker containers)
- `DB_OPERATION_TYPE`: LocalPasswordlessRoot (local), Azure credentials (prod)
- `SERVICE_NAME`: Unique service identifier
- `Otlp__Endpoint`: http://otel-lgtm:4317
- `OTEL_METRIC_EXPORT_INTERVAL`: 5000ms (5 seconds)

---

## Security & Secrets Management

### Azure Integration
- **Azure Key Vault**: Centralized secrets storage
- **Azure AD Service Principal**: Authentication for Azure resources
- Environment variables:
  - `AZURE_TENANT_ID`
  - `AZURE_CLIENT_ID`
  - `AZURE_CLIENT_SECRET`

### Database Security
- **LocalPasswordlessRoot**: Local development (Docker)
- **Azure Managed Identity**: Production deployment
- Connection string management via secrets

### API Authentication
- Provider-specific authentication (OAuth, API keys, certificates)
- Credential rotation support
- Secure credential storage in Key Vault

---

## Monitoring & Observability

### OpenTelemetry Integration
Every service includes:
- Metrics collection (custom and system metrics)
- Distributed tracing across service boundaries
- Structured logging
- Export to OTLP endpoint (port 4317)

### Metrics Export
- **Export Interval**: 5000ms (5 seconds)
- **Collector**: otel-lgtm container
- **Retention**:
  - Logs (Loki): 48 hours
  - Metrics (Mimir): 720 hours (30 days)
  - Traces (Tempo): 720 hours (30 days)

### Grafana Dashboard
- Port 3000 (when enabled)
- Unified dashboard for logs, metrics, traces
- Service-level dashboards
- Business metrics visualization

### Service Health
- HTTP health endpoints
- Container health checks
- Dependency health monitoring

---

## Development Guidelines

### Project Structure
```
scalar-mono/
├── Acquisition/                  # Data acquisition service
├── Ingestion/                   # Data ingestion service
├── Execution/                   # Trading execution service (framework only)
├── Signals/                     # Signal generation service (framework only)
├── Interface/                   # Web UI service (framework only)
├── Insights/                    # Python ML forecasting service
│   ├── source/                 # Python source code
│   │   ├── baltics_imbalance/  # Baltics forecasting
│   │   ├── dk_balancing_demand_forecast/  # DK demand
│   │   ├── dk_da_strategy/     # Day-ahead strategy
│   │   ├── DKAct_2/            # DK activation model
│   │   ├── shared/             # Shared utilities
│   │   │   ├── communication/  # Pulsar integration
│   │   │   ├── minitoring/     # OpenTelemetry
│   │   │   ├── persistence/    # Database config
│   │   │   └── scheduler/      # Forecast scheduling
│   │   └── utilities/          # Helper functions
│   ├── Dockerfile              # Python container build
│   └── docker-compose.yaml     # Separate deployment
├── Shared/                      # Shared .NET library
│   ├── Clients/                # External API clients
│   ├── Communication/          # Messaging utilities
│   ├── Models/                 # Data models
│   ├── Monitoring/             # Observability
│   ├── Persistence/            # Database access
│   └── Secrets/                # Secrets management
├── EliaIntegration/            # Elia provider integration
├── EntsoeClient/               # Entsoe client library
├── Scalar.DAL/                 # Data Access Layer
├── Scalar.Elia.Test/           # Elia tests
├── Scalar.EntsoeClient.Tests/  # Entsoe tests
├── Scalar.JAO.Client/          # JAO client library
├── Scalar.JAO.Test/            # JAO tests
├── Scalar.MessageBroker/       # Message broker abstractions
├── PulsarTransit/              # Pulsar utilities
├── PulsarTransit.Abstractions/ # Pulsar abstractions
├── Shared.UnitTests/           # Unit tests
├── Shared.IntegrationTests/    # Integration tests
├── Signals.UnitTests/          # Signals unit tests
├── Deployment/                 # Deployment scripts
├── build-deps/                 # External dependencies
├── docker-compose.yaml         # Main container orchestration
└── Scalar.sln                  # Solution file
```

### Service Development Pattern
1. Create worker class inheriting from `BackgroundService`
2. Inject required dependencies (clients, transactions, config)
3. Register as `IHostedService` in `Program.cs`
4. Implement data processing logic
5. Add monitoring and logging
6. Write unit and integration tests

### Adding New Data Provider
1. **Create dedicated client library** (e.g., `Scalar.{Provider}.Client/`)
   - Or add to `Shared/Clients/{Provider}/` for simpler providers
2. **Define models** in `Shared/Models/{Provider}/`
3. **Create persistence layer** in `Shared/Persistence/{Provider}/`
4. **Implement acquisition workers** in `Acquisition/Workers/{Provider}/`
   - Use `TimeWindowedEntsoeWorkerBase` pattern for time-series data
   - Configure separate authentication tokens per endpoint if needed
5. **Implement ingestion workers** in `Ingestion/Workers/{Provider}/`
6. **Register services** in respective `Program.cs` files
   - Use environment-based conditional registration (Development vs Production)
7. **Add configuration seeding** for worker enable/disable control
8. **Add tests** in test projects (create `Scalar.{Provider}.Test/` if needed)

### Testing Strategy
- **Unit Tests**: Business logic, transformations, calculations
- **Integration Tests**: Database operations, API clients, message broker
- **Component Tests**: Service-to-service communication
- **End-to-End Tests**: Complete data flow validation

### Build & Deployment
```bash
# Local development
dotnet build Scalar.sln

# Docker build
docker-compose build

# Run locally
docker-compose up -d

# View logs
docker-compose logs -f [service-name]

# Stop services
docker-compose down
```

### Database Migration Pattern
- Scripts managed in `Shared/Persistence/SQL/`
- Version-controlled migration scripts
- Automatic execution on startup (development)
- Manual execution in production

---

## Component Interactions

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│  External   │      │ Acquisition │      │   Pulsar    │
│  Providers  │─────▶│   Service   │─────▶│   Broker    │
│             │      │ (40+ workers)│      │   4.0.5     │
└─────────────┘      └─────────────┘      └─────────────┘
  Elia (15+)                                      │
  Entsoe (9)                                      ▼
  JAO (6)            ┌─────────────┐      ┌─────────────┐
  Nordpool           │ StarRocks   │◀─────│  Ingestion  │
  Energinet          │   OLAP DB   │      │   Service   │
  Meteologica        │    3.4.0    │      │ (19+ workers)│
  Volue              └─────────────┘      └─────────────┘
                            │                    │
                            │             ┌─────────────┐
                            │             │   Redis     │
                            │             │   Cache     │
                            │             │  7.4.0-v0   │
                            │             └─────────────┘
                            │                    │
                            └────────────┬───────┘
                                         ▼
                                  ┌─────────────┐
                                  │  Insights   │
                                  │  Service    │
                                  │  (Python)   │──▶ Pulsar
                                  └─────────────┘    (Forecasts)
                                  ML Forecasts
                                         │
                            ┌────────────┴────────────┐
                            ▼                         ▼
                     ┌─────────────┐          ┌─────────────┐
                     │  Execution  │          │  Interface  │
                     │   Service   │          │   Service   │
                     │  (planned)  │          │  (planned)  │
                     └─────────────┘          └─────────────┘
```

---

## Current Implementation Status

### ✅ Production Ready
- **Acquisition Service**: 40+ workers across 7+ providers
  - Elia (15+ workers) - Primary provider
  - Entsoe (9 workers) - Full integration
  - JAO (6 workers) - Complete
  - Nordpool (realtime + rest)
  - Energinet, Meteologica, Volue
- **Ingestion Service**: 19+ workers with batch processing
- **Insights Service**: Python ML forecasting service
  - Nordic imbalance forecasting
  - Baltics imbalance predictions
  - DK balancing demand forecasts
  - Day-ahead strategy generation
- **Shared library infrastructure**: Complete
- **Docker containerization**: Multi-service orchestration
- **LGTM observability stack**: Active (Loki, Grafana, Tempo, Mimir)
- **Database infrastructure**: StarRocks 3.4, Redis 7.4, Pulsar 4.0.5
- **Message broker**: Apache Pulsar with full BookKeeper persistence

### 🚧 Framework Ready (Not Deployed)
- Signals Service (minimal skeleton)
- Execution Service (minimal skeleton)
- Interface Service (Blazor framework ready)

### 📋 Planned
- Activate Signals and Execution services
- Deploy Interface service for visualization
- Advanced signal algorithms beyond ML forecasts
- Full automated trading strategies
- Horizontal scaling via Pulsar partitioning
- Multi-region deployment

---

## Key Design Decisions

1. **Monorepo Structure**: All services in single repository for easier coordination
2. **Polyglot Architecture**: .NET for core services, Python for ML workloads
3. **Shared Library**: Common code centralized to avoid duplication
4. **Dedicated Provider Libraries**: Major providers (Elia, Entsoe, JAO) have separate client libraries
5. **Message-Driven**: Asynchronous communication via Apache Pulsar for scalability and resilience
6. **StarRocks for Analytics**: OLAP database optimized for time-series queries
7. **Redis for Real-Time**: Fast access to current market state
8. **Worker Pattern**: Background services for continuous data processing
9. **Configuration-Driven Activation**: Workers can be enabled/disabled via database seeding
10. **Environment-Specific Workers**: Different providers active in Development vs Production
11. **Batch Processing**: Optimized bulk inserts for high-throughput ingestion
12. **Observability Built-In**: OpenTelemetry + LGTM stack as first-class concern
13. **Azure-First Secrets**: Production secrets managed in Azure Key Vault
14. **Containerized Deployment**: Consistent environments across dev/prod
15. **Separate ML Service**: Python Insights service runs independently for ML forecasting

---

## Scalability Considerations

- **Horizontal Scaling**: Services can be replicated behind load balancer
- **Message Partitioning**: Pulsar topics can be partitioned by provider/data type
- **Pulsar BookKeeper**: Persistent ledger storage for message durability
- **Batch Acknowledgement**: Pulsar batch index-level acknowledgement enabled
- **Database Sharding**: StarRocks supports distributed query execution
- **Cache Distribution**: Redis can be clustered for high availability
- **Rate Limiting**: Polly policies protect external API quotas
- **Backpressure Handling**: Pulsar consumer acknowledgment controls flow
- **Resource Limits**: Memory and CPU limits configured per service
- **Worker Isolation**: 40+ acquisition workers operate independently

---

## Future Enhancements

1. **Multi-Region Deployment**: Geographic distribution for resilience
2. **Enhanced ML Models**: Expanded forecasting coverage across all markets
3. **Real-Time Signal Processing**: Activate Signals service for live analytics
4. **Automated Execution**: Deploy Execution service for strategy automation
5. **API Gateway**: Unified external API with authentication
6. **Event Sourcing**: Complete audit trail of all data changes
7. **GraphQL Interface**: Flexible data querying for frontend
8. **Kubernetes Deployment**: Production-grade orchestration
9. **Advanced Monitoring**: Anomaly detection and alerting
10. **Model Versioning**: ML model registry and A/B testing

---

## Recent Architectural Changes

### December 2025
- **Schema Evolution**: Removed direction column from system imbalance data (commit 8c1447e)
- **Token Management**: Entsoe workers refactored to use separate tokens per API endpoint (commit bfaf984)
- **Worker Configuration**: Added database-driven worker enable/disable capability (commit 96be402)
- **Pulsar Optimization**: Increased memory allocations for broker, zookeeper, and bookie (commit 843d1da)

### November 2025
- **Insights Service**: Added Python ML forecasting service with Nordic and Baltics models
- **Elia Integration**: Completed 15+ worker implementation for Belgian TSO
- **JAO Integration**: Implemented 6 workers for Joint Allocation Office data
- **Time-Windowed Workers**: Introduced `TimeWindowedEntsoeWorkerBase` abstraction for better data retrieval patterns

---

**Document Version**: 2.0
**Last Updated**: December 11, 2025
**Maintained By**: Scalar Development Team
