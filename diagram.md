```mermaid
flowchart LR
  %% --- Edge / Ingress ---
  U[Users (Web/Mobile)] -->|HTTPS| CDN[CDN + WAF]
  CDN --> AG[API Gateway / AuthN+AuthZ]
  AG --> INGRESS[Ingress Controller (Nginx/Envoy)]

  %% --- App cluster ---
  subgraph AppCluster [Kubernetes App Cluster]
    INGRESS --> WEB[Web UI (Next.js/SPA)]
    INGRESS --> API[API (REST/GraphQL)]
    INGRESS --> WS[Realtime Hub (WS/SSE)]
    API --> REDIS[(Redis Cache)]
    API --> S3[(Object Storage)]
    API --> PG[(Postgres Primary)]
    PG --> PGRO[(Read Replicas)]
  end

  %% --- Event pipeline ---
  SRPC[(Soroban RPC)]
  INDEX[Indexers (pollers)]
  KAFKA[(Kafka / Redpanda)]
  WRK[Workers (Rust consumers)]
  MV[(Materialized Views)]

  %% --- Connections ---
  INDEX -->|egress: getEvents| SRPC
  INDEX -->|produce| KAFKA
  KAFKA --> WRK
  WRK -->|upsert| PG
  WRK -->|refresh/compute| MV
  MV --> REDIS

  %% --- Read paths ---
  API --> REDIS
  API --> MV
  API --> PGRO

```
