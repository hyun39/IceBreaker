# RAG Architecture Diagram

## 1. Diagram Type & Rationale

`flowchart TD` (top-down) with subgraphs — best fit because RAG has a clear directional pipeline, and subgraphs let us visually separate the **offline indexing path** from the **online query path** while showing the shared Vector Database connection.

---

## 2. Mermaid Diagram

```mermaid
flowchart TD
    %% ─────────────────────────────────────────
    %% INDEXING PIPELINE  (offline / pre-processing)
    %% ─────────────────────────────────────────
    subgraph INDEXING["Indexing Pipeline (Offline / Pre-processing)"]
        direction TB
        A1["Raw Documents\n(PDFs, web pages, docs, etc.)"]
        A2["Chunker\n(split into overlapping passages)"]
        A3["Embedding Model\n(e.g. text-embedding-3-small)"]
        A4[("Vector Database\n(e.g. Pinecone, Chroma, FAISS)")]

        A1 --> A2
        A2 -->|"Chunk texts"| A3
        A3 -->|"Dense vectors\n(stored with chunk text)"| A4
    end

    %% ─────────────────────────────────────────
    %% QUERY PIPELINE  (online / real-time)
    %% ─────────────────────────────────────────
    subgraph QUERYING["Query Pipeline (Online / Real-time)"]
        direction TB
        B1["User Query\n(natural language question)"]
        B2["Query Encoder\n(same Embedding Model)"]
        B3["Similarity Search\n(ANN / cosine similarity)"]
        B4["Retrieved Document Chunks\n(top-k most relevant passages)"]

        B1 -->|"Raw query text"| B2
        B2 -->|"Query vector"| B3
        B3 -->|"Fetch top-k chunks\nfrom vector DB"| B4
    end

    %% ─────────────────────────────────────────
    %% GENERATION  (online)
    %% ─────────────────────────────────────────
    subgraph GENERATION["Generation"]
        direction TB
        C1["Prompt Assembler\n(Prompt Template + Query + Context)"]
        C2["LLM / Generator\n(e.g. GPT-4o, Claude, Llama 3)"]
        C3["Final Response\n(grounded, cited answer)"]

        C1 -->|"Fully constructed prompt"| C2
        C2 -->|"Generated text"| C3
    end

    %% ─────────────────────────────────────────
    %% CROSS-SUBGRAPH CONNECTIONS
    %% ─────────────────────────────────────────
    A4 -->|"Index ready"| B3
    B4 -->|"Inject context"| C1
    B1 -->|"Original question\n(passed through)"| C1

    %% ─────────────────────────────────────────
    %% STYLING
    %% ─────────────────────────────────────────
    classDef offline  fill:#e8f4f8,stroke:#4a9aba,stroke-width:2px,color:#1a3a4a
    classDef online   fill:#f0faf0,stroke:#4aaa6a,stroke-width:2px,color:#1a3a1a
    classDef generate fill:#fdf6e3,stroke:#c9a227,stroke-width:2px,color:#3a2a00
    classDef db       fill:#f5e6ff,stroke:#8a44bb,stroke-width:2px,color:#2a0050
    classDef output   fill:#fff0f0,stroke:#cc4444,stroke-width:2px,color:#4a0000

    class A1,A2,A3 offline
    class A4 db
    class B1,B2,B3,B4 online
    class C1,C2 generate
    class C3 output
```

---

## 3. Key Elements Explained

| Component | Role |
|---|---|
| **Embedding Model** | Shared linchpin — used in *both* pipelines. Chunks and queries must be in the same vector space for similarity search to work. |
| **Vector Database** | The only stateful component. Stores vectors + source text + metadata. Everything else is stateless per request. |
| **Chunker** | Splits docs into overlapping passages (~512 tokens) so each piece is semantically coherent. |
| **Similarity Search** | ANN (Approximate Nearest Neighbor) search via cosine similarity — finds the top-k passages closest to the query vector. |
| **Prompt Assembler** | Where the "Augmentation" in RAG actually happens — injects retrieved context into the prompt template alongside the original query. |
| **User Query (dual path)** | The original query takes **two independent paths**: one encoded for retrieval, one raw for the prompt — both are needed. |

---

## 4. Optional Extensions

- **Reranker** — insert a `cross-encoder reranker` node between `Similarity Search` and `Retrieved Chunks` for production precision
- **Query Rewriter** — add a `Query Rewriter (LLM)` node before the encoder (HyDE, multi-query, step-back prompting)
- **Output Guardrails** — add a `Safety / Hallucination Filter` between the LLM and Final Response for enterprise use cases
- **Alternative view** — use `sequenceDiagram` if you want to show latency and request/response timing across named actors (User ↔ Vector DB ↔ LLM)
