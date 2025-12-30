## System Architecture

```mermaid
flowchart TB
  subgraph EDGE["Edge - User - Docker"]
    EEG["EEG Headset"] --> C["client-eeg-streamer"]
    C --> MCP_S["mcp-tool-server<br/>IoT + Media Tools"]
    UI["React UI"] --> UIB["ui-backend"]
  end

  subgraph INGRESS["Ingress"]
    NGINX["nginx-ws-gateway<br/>WSS + WS routing"]
  end

  subgraph CLOUD["GKE - Dockerized Services"]
    GW["api-gateway"]
    ING["stream-ingest"]
    BUF[("Pub/Sub")]
    FE["feature-extractor"]
    INF["inference-service<br/>GPU pool"]
    AG["assistant-agent<br/>RAG + Canvas + MCP"]
    RAG["rag-retriever<br/>course-aware"]
    VDB[("Vector DB / Index<br/>per-course collections")]
    CAN["Canvas Tool Connector"]
  end

  %% Streams
  C -->|"WSS /ws/eeg"| NGINX --> GW --> ING --> BUF --> FE --> INF

  %% UI
  UIB -->|"WS /ws/ui"| NGINX
  INF --> UIB

  %% Agent
  INF --> AG
  AG --> RAG --> VDB
  AG --> CAN
  AG <-->|"MCP"| MCP_S
  AG --> UIB
```

## Training Architecture

```mermaid
flowchart LR
  subgraph DATA["Data Lake + Versioning"]
    REC["stream-to-parquet"] --> GCS[("GCS: Parquet partitions")]
    GCS --> VER["dataset-versioner<br/>manifests + snapshots"]
    VER --> SNAP[("GCS/Git: dataset snapshots")]
    GCS --> ETL["etl-to-bigquery<br/>clean + standardize"]
    ETL --> BQ[("BigQuery: training tables")]
  end

  subgraph TRAIN["Training + Finetuning"]
    SNAP --> TR["trainer-vertex<br/>submit jobs"]
    BQ --> TR
    TR --> VAI["Vertex AI Training<br/>train + finetune"]
    VAI --> ART[("GCS: model artifacts")]
    VAI --> WB["Weights and Biases<br/>metrics + artifacts + registry"]
    ART --> WB
  end

  subgraph SERVE["Serving"]
    ART --> INF["inference-service pulls model"]
  end
```


### Key interfaces (so engineering stays crisp)

#### Edge → Cloud

- Protocol: WebSockets or gRPC streaming

- Payload: {session_id, timestamp, channel_samples[], sampling_rate, impedance?, quality_flags}

#### Cloud → UI

- Streaming: WebSocket

- Payload: {text, confidence, token_timestamps, is_command?, tool_results?}

#### Inference boundary

- Feature windowing: e.g., 250–1000 ms windows, sliding stride

- Output rate: ~2–5 Hz "partial decode" updates + finalized tokens

---
# Deployment 

## 1 - Deployment Diagram (GCP + GKE + Autoscaling + RAG + Canvas)

```mermaid
flowchart TB
  subgraph EDGE["Edge - Docker"]
    EEG["EEG Headset"] --> CL["client-eeg-streamer"]
    MCP["mcp-tool-server<br/>IoT+Media"] <-->|"MCP"| CL
    UI["React UI"] --> UIB["ui-backend"]
  end

  subgraph GCP["GCP Managed"]
    GCS[("Cloud Storage<br/>Parquet + Artifacts")]
    BQ[("BigQuery")]
    VERTEX["Vertex AI"]
    PUB["Pub/Sub"]
    SM["Secret Manager"]
    MON["Cloud Monitoring/Logging"]
    VDB[("Vector DB<br/>Vertex Matching Engine / Pinecone / Weaviate")]
  end

  subgraph GKE["GKE - HPA/KEDA"]
    NGINX["nginx-ws-gateway"]
    GW["api-gateway<br/>HPA"]
    ING["stream-ingest<br/>KEDA/HPA"]
    FE["feature-extractor<br/>HPA"]
    INF["inference-service<br/>GPU pool + HPA"]
    AG["assistant-agent<br/>HPA"]
    RAG["rag-retriever<br/>HPA"]
    CAN["canvas-connector<br/>HPA"]
    REC["stream-to-parquet<br/>Job"]
    ETL["etl-to-bigquery<br/>CronJob/Job"]
    VER["dataset-versioner<br/>Job"]
  end

  CL -->|"WSS"| NGINX --> GW --> ING --> PUB --> FE --> INF --> UIB
  INF --> AG --> RAG --> VDB
  AG --> CAN
  AG <-->|"MCP"| MCP

  REC --> GCS --> ETL --> BQ
  GCS --> VER
  BQ --> VERTEX --> GCS
  GCS --> INF

  SM --> GW
  SM --> AG
  SM --> CAN
  GW --> MON
  INF --> MON
  AG --> MON
```
## GitHub Actions Workflow (High-Level)

```yaml
name: ci

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build containers
        run: docker compose build
      - name: Run unit tests
        run: docker compose run tests
      - name: Run integration tests
        run: docker compose run integration-tests
      - name: Coverage check
        run: ./scripts/check_coverage.sh

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Build & push images
        run: ./scripts/build_and_push.sh

```

