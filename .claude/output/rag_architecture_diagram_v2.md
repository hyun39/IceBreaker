# RAG Architecture Diagram v2 (with ASCII Concept Map)

## Concept Summary

**RAG in one sentence:** Store knowledge as searchable vectors → at query time, find the most relevant chunks → stuff them into the LLM prompt so it answers *from your data*, not just its training.

```
OFFLINE (once)                  ONLINE (per query)
──────────────────              ────────────────────────────────────────
[Docs]                          [User Question]
  │                                │              │
  ▼                                ▼              │ (passed through raw)
[Chunk]                         [Embed]           │
  │                                │              │
  ▼                                ▼              │
[Embed] ──vectors──► [VecDB] ◄─── search          │
                         │                        │
                         ▼                        │
                    [Top-k Chunks] ───────► [Prompt Assembler]
                                                  │
                                                  ▼
                                               [LLM]
                                                  │
                                                  ▼
                                              [Answer]
```

---

## 1. Diagram Type & Rationale

`flowchart TD` with three subgraphs — mirrors the ASCII skeleton exactly: offline flows into online, online flows into generation. The phase boundaries become visible fences.

---

## 2. Mermaid Diagram

```mermaid
flowchart TD

    %% ── OFFLINE INDEXING PIPELINE ──────────────────────────────────
    subgraph OFFLINE["Indexing Pipeline (Offline — runs once)"]
        direction TB
        A["Raw Documents\n(PDFs, HTML, text files, databases)"]
        B["Chunker\n(Split docs into overlapping passages)"]
        C["Embedding Model\n(Converts text → dense float vectors)"]
        D[("Vector Database\n(Stores & indexes all chunk vectors)")]

        A --> B
        B --> C
        C -->|"store vectors\n+ metadata"| D
    end

    %% ── ONLINE QUERY PIPELINE ───────────────────────────────────────
    subgraph ONLINE["Query Pipeline (Online — per user request)"]
        direction TB
        E["User Query\n(Natural language question)"]
        F["Query Encoder\n(Same Embedding Model as indexing)"]
        G["Similarity Search\n(ANN lookup: cosine / dot-product)"]
        H["Retrieved Chunks\n(Top-k most relevant passages)"]

        E --> F
        F -->|"query vector"| G
        G -->|"ranked results"| H
    end

    %% ── GENERATION ──────────────────────────────────────────────────
    subgraph GEN["Generation"]
        direction TB
        I["Prompt Assembler\n(Injects query + chunks into template)"]
        J["LLM\n(Reads context, reasons, generates text)"]
        K["Final Response\n(Grounded, citation-ready answer)"]

        I --> J --> K
    end

    %% ── CROSS-SUBGRAPH EDGES ────────────────────────────────────────
    D -->|"indexed vectors\navailable for search"| G
    H -->|"retrieved context"| I
    E -->|"raw question\n(unchanged)"| I

    %% ── STYLE CLASSES ───────────────────────────────────────────────
    classDef offline    fill:#1e3a5f,stroke:#4a90d9,color:#cce4ff,rx:6
    classDef vecdb      fill:#3b1f6e,stroke:#9b59b6,color:#e8d5ff,rx:6
    classDef online     fill:#1a3d2b,stroke:#27ae60,color:#c8f7d8,rx:6
    classDef generation fill:#3d2a00,stroke:#e67e22,color:#ffe8b0,rx:6
    classDef output     fill:#4a1010,stroke:#e74c3c,color:#ffd5d5,rx:6

    class A,B,C offline
    class D vecdb
    class E,F,G,H online
    class I,J generation
    class K output
```

---

## 3. Component Table

| Component | Role |
|---|---|
| **Raw Documents** | Source corpus — any text you want the LLM to cite from. |
| **Chunker** | Splits long docs into focused passages (256–512 tokens, with overlap) so vectors stay semantically tight. |
| **Embedding Model** | Shared linchpin — encodes both chunks *and* queries into the same vector space so similarity search is meaningful. |
| **Vector Database** | The only stateful component; persists vectors + metadata and serves ANN search at query time. |
| **User Query** | Takes two parallel paths: one encoded (retrieval), one raw (prompt) — both are required. |
| **Similarity Search** | Cosine / dot-product ANN lookup — returns the top-k chunks closest to the query vector. |
| **Prompt Assembler** | Where "Augmentation" happens — merges the raw question + retrieved chunks into a structured LLM prompt. |
| **LLM** | Reads the assembled context and generates a grounded answer rather than relying solely on training weights. |
| **Final Response** | Traceable back to source passages — not hallucinated from model weights alone. |

---

## 4. Three Extensions to Add Next

1. **Re-Ranker** — between Similarity Search → Retrieved Chunks; a cross-encoder (Cohere Rerank, BGE) re-scores candidates more precisely than ANN distance
2. **Query Rewriter** — before Query Encoder; a small LLM rephrases/expands the query (HyDE, multi-query) to improve recall when user phrasing differs from source docs
3. **Faithfulness Check** — after LLM; an NLI model or second LLM call verifies each claim is supported by the retrieved chunks before the answer reaches the user
