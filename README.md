# R2R System Architecture Analysis

Based on the codebase and configuration analysis, here involves the system architecture and data flow of R2R.

## Core Components

1.  **R2R API Server (Core)**
    *   **Role**: Central orchestration and entry point.
    *   **Implementation**: Python (FastAPI/Uvicorn), structured via `py/core/main`.
    *   **Key Services**: 
        *   `Auth Service`: User authentication.
        *   `Ingestion Service`: Document processing pipeline.
        *   `Retrieval Service`: Search (Vector + Hybrid) and RAG generation.
        *   `Graph Service`: Knowledge graph management.

2.  **Infrastructure & Storage**
    *   **Postgres (pgvector)**: Primary database for relational data and vector embeddings.
    *   **Minio**: S3-compatible object storage for raw files.
    *   **Hatchet**: Workflow engine for distributed, asynchronous task processing (crucial for heavy ingestion jobs).
    *   **RabbitMQ**: Message broker used by Hatchet.

3.  **Specialized Workers**
    *   **Unstructured**: Service for parsing complex document formats (PDF, PPT, etc.).
    *   **Graph Clustering**: Dedicated service for graph operations.

## System Flow Diagram

```mermaid
graph TD
    User[User / Client] -->|HTTP Request| API[R2R API Server]
    
    subgraph Services [Core Services]
        API --> AuthS[Auth Service]
        API --> IngestS[Ingestion Service]
        API --> RetrS[Retrieval Service]
        API --> GraphS[Graph Service]
    end

    subgraph Infrastructure [Data & Infrastructure]
        DB[(Postgres\nVector + Relational)]
        Minio[(Minio S3\nFile Storage)]
        Hatchet[Hatchet Workflow Engine]
        MQ[RabbitMQ]
        Worker[R2R Worker]
        Unstructured[Unstructured Service]
    end

    subgraph External [External LLM Providers]
        LLM[OpenAI / Anthropic / Etc.]
    end

    %% Ingestion Workflow
    IngestS -->|1. Upload File| Minio
    IngestS -->|2. Dispatch Job| Hatchet
    Hatchet -->|3. Queue Job| MQ
    MQ -->|4. Consume Job| Worker
    
    Worker -->|5. Parse Document| Unstructured
    Worker -->|6. Generate Embeddings| LLM
    Worker -->|7. Store Vectors & Chunks| DB
    Worker -->|8. Extract Entities (Graph)| LLM
    Worker -->|9. Update Graph| DB

    %% Retrieval Workflow
    RetrS -->|1. Search Query| LLM
    RetrS -.->|Embedding| LLM
    RetrS -->|2. Hybrid Search| DB
    RetrS -->|3. Knowledge Graph Search| DB
    RetrS -->|4. RAG Generation| LLM
    
    %% Auth
    AuthS -->|Verify| DB
```

## Workflow Descriptions

### 1. Ingestion Flow (Asynchronous)
1.  **Upload**: User sends a file to the API.
2.  **Storage**: System saves the raw file to Minio.
3.  **Orchestration**: `Ingestion Service` triggers a Hatchet workflow.
4.  **Processing**:
    *   Workers pick up the job.
    *   `Unstructured` service parses the file content.
    *   Text is chunked and embedded via External LLMs.
    *   Vectors are stored in Postgres (`pgvector`).
    *   Knowledge Graph triples are extracted and stored if enabled.

### 2. Retrieval / RAG Flow (Synchronous)
1.  **Query**: User sends a natural language query.
2.  **Search**:
    *   Query is embedded.
    *   Hybrid search (Semantic + Keyword) runs against Postgres.
    *   Graph search navigates relationships in the DB.
3.  **Generation**:
    *   Retrieved context is assembled.
    *   LLM generates the final response based on the context.
