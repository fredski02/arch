```mermaid
flowchart LR
  %% ==== EDGE / INGRESS ====
  subgraph EDGE[Edge / Ingress]
    U[Users (Web/Mobile)]
    CDN[CDN + WAF]
    AG[API Gateway / AuthN+AuthZ]
    U -->|HTTPS| CDN --> AG
  end

  %% ==== APP CLUSTER ====
  subgraph K8S[Kubernetes App Cluster]
    INGRESS[Nginx/Envoy Ingress Controller]
    WEB[Web UI (Next.js/SPA)]
    API[API (REST/GraphQL)]
    WS[Realtime Hub (WS/SSE)]
    REDIS[(Redis Cache)]
    O11Y[Telemetry Agents\n(OpenTelemetry / Promtail)]
    S3[(Object Storage\n(snapshots/exports))]

    AG --> INGRESS
    INGRESS --> WEB
    INGRESS --> API
    INGRESS --> WS

    API --> REDIS
    API -->|read/write| S3
    API --> O11Y
    WEB --> O11Y
    WS --> O11Y

    %% Indexers live in app cluster but egress-only to RPC
    INDEX[Indexers (Soroban → Kafka producers)]
    INDEX --> O11Y
  end

  %% ==== DATA PLANE (PRIVATE) ====
  subgraph DATA[Data Plane (Private Network)]
    SRPC[(Soroban RPC\n(public or self-hosted))]
    KAFKA[(Kafka / Redpanda)]
    SCHEMA[(Schema Registry)]
    PG[(Postgres Primary)]
    PGRO[(Postgres Read Replicas)]
    MATVIEWS[(Materialized Views)]
    BACKUP[(Backups / WAL Archive)]
    REDIS_D[(Redis Cluster)]

    KAFKA --- SCHEMA
    PG --> PGRO
    PG --> BACKUP
    MATVIEWS --> REDIS_D
  end

  %% ==== WORKERS / INGEST ====
  INDEX -- filtered getEvents --> SRPC
  INDEX -->|produce| KAFKA
  WRK[Workers (Rust consumers)]
  WRK -->|upserts| PG
  WRK -->|refresh/compute| MATVIEWS
  WRK -->|hot aggregates| REDIS_D

  %% ==== APP READ PATHS ====
  API -->|hot reads| REDIS_D
  API -->|materialized reads| MATVIEWS
  API -->|strong reads| PGRO

  %% ==== SECURITY NOTES ====
  classDef edge fill:#f5faff,stroke:#5b8def,stroke-width:1px;
  classDef app fill:#f8fff5,stroke:#58b368,stroke-width:1px;
  classDef data fill:#fff8f0,stroke:#ffa24a,stroke-width:1px;

  class EDGE edge
  class K8S app
  class DATA data

  %% Connectivity hints
  K8S ---|PrivateLink/VPC Peering| DATA
```
