# RAG (Retrieval-Augmented Generation) Architecture Diagram

```mermaid
flowchart TD
    U([👤 User]) -->|"Natural language query"| Q[Query]

    subgraph INGESTION["📥 Offline: Knowledge Base Ingestion"]
        direction LR
        D[(Raw Documents\nPDFs · Web · DB)] --> CH[Chunking\nSplitter]
        CH --> EMB_I[Embedding Model]
        EMB_I -->|"Dense vectors"| VDB[(Vector\nDatabase)]
    end

    subgraph RETRIEVAL["🔍 Online: Retrieval Phase"]
        direction TB
        Q --> EMB_Q[Query Embedding\nModel]
        EMB_Q -->|"Query vector"| SIM["Similarity Search\n(cosine / dot product)"]
        VDB -->|"Candidate vectors"| SIM
        SIM --> RANK["Ranking &\nReranking"]
        RANK --> CHUNKS["Top-K Relevant\nDocument Chunks"]
    end

    subgraph GENERATION["🤖 Online: Generation Phase"]
        direction TB
        CHUNKS --> CTX["Context\nPreparation\n(Prompt Assembly)"]
        Q --> CTX
        CTX -->|"System prompt +\ncontext + query"| LLM["Large Language\nModel\nGPT / Claude / etc."]
        LLM --> ANS["Generated\nResponse"]
    end

    ANS --> U

    style INGESTION fill:#1e3a5f,stroke:#4a90d9,color:#e8f4fd
    style RETRIEVAL fill:#1a3a2a,stroke:#4caf7d,color:#e8f9f0
    style GENERATION fill:#3a1e1e,stroke:#e07b54,color:#fdf0ec
    style U fill:#4a4a6a,stroke:#9090c0,color:#ffffff
    style VDB fill:#2a5580,stroke:#4a90d9,color:#ffffff
```

## Phase Summary

| Phase | Description |
|-------|-------------|
| **Ingestion** | Split documents into chunks → generate embedding vectors → store in Vector DB (offline) |
| **Query Embedding** | Vectorize user query using the same embedding model |
| **Similarity Search** | Calculate cosine similarity between query vector and DB vectors |
| **Reranking** | Re-order Top-K chunks by relevance score |
| **Context Prep** | Assemble system prompt + retrieved chunks + original query into a single prompt |
| **LLM Generation** | Generate a grounded, accurate response using the assembled context |
