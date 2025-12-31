---
title: Architecture Documentation
description: System architecture, components, data flow, and design decisions for the microwave RAG assistant
version: 1.0.0
last_updated: 2025-12-30
related: [README.md, api.md, adr/]
tags: [architecture, design, rag-pipeline, components]
---

# Architecture Documentation

## Table of Contents
- [System Overview](#system-overview)
- [Component Architecture](#component-architecture)
- [Data Flow](#data-flow)
- [Module Breakdown](#module-breakdown)
- [Design Decisions](#design-decisions)
- [Integration Points](#integration-points)
- [Constraints & Trade-offs](#constraints--trade-offs)

## System Overview

The AI DIAL RAG Overview implements a classic three-stage RAG architecture for question-answering over a domain-specific knowledge base (microwave manual).

### Architecture Diagram

```mermaid
graph TD
    subgraph "Application Layer"
        A[main function] --> B[MicrowaveRAG]
        B --> C[AzureOpenAIEmbeddings]
        B --> D[AzureChatOpenAI]
    end
    
    subgraph "RAG Pipeline"
        B --> E[_setup_vectorstore]
        E --> F{Index Exists?}
        F -->|Yes| G[FAISS.load_local]
        F -->|No| H[_create_new_index]
        H --> I[TextLoader]
        H --> J[RecursiveCharacterTextSplitter]
        H --> K[FAISS.from_documents]
        K --> L[vectorstore.save_local]
        
        B --> M[retrieve_context]
        M --> N[similarity_search_with_relevance_scores]
        
        B --> O[augment_prompt]
        O --> P[Format prompt template]
        
        B --> Q[generate_answer]
        Q --> R[LLM.generate]
    end
    
    subgraph "External Services"
        C --> S[DIAL API - Embeddings]
        D --> T[DIAL API - Chat Completion]
    end
    
    subgraph "Data Storage"
        U[microwave_manual.txt]
        V[microwave_faiss_index/]
    end
    
    I --> U
    L --> V
    G --> V
    
    style B fill:#e1f5ff
    style E fill:#fff4e1
    style M fill:#ffe1f5
    style O fill:#ffe1f5
    style Q fill:#e1ffe1
```

### System Characteristics

| Characteristic | Value |
|---------------|-------|
| **Architecture Pattern** | Pipeline (3-stage RAG) |
| **Deployment Model** | Single-process CLI application |
| **State Management** | Stateless (per-query), persistent vector index |
| **Concurrency** | Single-threaded, synchronous |
| **Scalability** | Vertical (constrained by FAISS in-memory index) |
| **Data Persistence** | Local filesystem (FAISS index) |

## Component Architecture

### 1. MicrowaveRAG Class

**Responsibility**: Orchestrates the complete RAG pipeline

**Dependencies**:
- `AzureOpenAIEmbeddings`: Converts text to vectors
- `AzureChatOpenAI`: Generates answers from prompts
- `FAISS`: Vector similarity search

**Key Methods**:

```mermaid
classDiagram
    class MicrowaveRAG {
        -AzureOpenAIEmbeddings embeddings
        -AzureChatOpenAI llm_client
        -VectorStore vectorstore
        +__init__(embeddings, llm_client)
        +retrieve_context(query, k, score) str
        +augment_prompt(query, context) str
        +generate_answer(augmented_prompt) str
        -_setup_vectorstore() VectorStore
        -_create_new_index() VectorStore
    }
    
    class AzureOpenAIEmbeddings {
        +embed_query(text) List~float~
        +embed_documents(texts) List~List~float~~
    }
    
    class AzureChatOpenAI {
        +generate(messages) LLMResult
    }
    
    class FAISS {
        +similarity_search_with_relevance_scores(query, k, threshold)
        +load_local(path, embeddings)
        +from_documents(docs, embeddings)
        +save_local(path)
    }
    
    MicrowaveRAG --> AzureOpenAIEmbeddings
    MicrowaveRAG --> AzureChatOpenAI
    MicrowaveRAG --> FAISS
```

### 2. Configuration Module (`_constants.py`)

**Responsibility**: Centralized configuration management

```python
DIAL_URL: str          # DIAL API endpoint
API_KEY: str           # Authentication token (from env)
```

**Design Note**: Environment variable fallback pattern ensures secrets aren't committed. See [ADR-002: Environment-Based Configuration](./adr/ADR-002-environment-config.md).

### 3. Knowledge Base (`microwave_manual.txt`)

**Format**: Plain text (UTF-8)  
**Size**: ~497 lines, ~25KB  
**Structure**: Sections on safety, operation, cleaning, specifications

**Preprocessing**: None (loaded as-is, chunked by `RecursiveCharacterTextSplitter`)

### 4. Vector Index (`microwave_faiss_index/`)

**Format**: FAISS binary index  
**Location**: `./microwave_faiss_index/index.faiss`  
**Generation**: Lazy (created on first run if missing)  
**Persistence**: Local filesystem

**Index Properties**:
- **Embedding Model**: `text-embedding-3-small-1` (Azure OpenAI)
- **Dimensionality**: 1536 (determined by embedding model)
- **Distance Metric**: L2 (Euclidean) [FAISS default]

## Data Flow

### End-to-End Query Flow

```mermaid
sequenceDiagram
    actor User
    participant Main
    participant RAG as MicrowaveRAG
    participant VS as VectorStore
    participant Embed as Embeddings
    participant LLM as AzureChatOpenAI
    participant DIAL as DIAL API
    
    User->>Main: Enter query
    Main->>RAG: retrieve_context(query)
    
    Note over RAG,VS: STAGE 1: RETRIEVAL
    RAG->>Embed: embed_query(query)
    Embed->>DIAL: POST /embeddings
    DIAL-->>Embed: embedding vector
    Embed-->>RAG: query_embedding
    
    RAG->>VS: similarity_search_with_relevance_scores(query_embedding, k=4)
    VS-->>RAG: [(doc, score), ...]
    RAG-->>Main: context string
    
    Main->>RAG: augment_prompt(query, context)
    Note over RAG: STAGE 2: AUGMENTATION
    RAG->>RAG: Format USER_PROMPT template
    RAG-->>Main: augmented_prompt
    
    Main->>RAG: generate_answer(augmented_prompt)
    Note over RAG,LLM: STAGE 3: GENERATION
    RAG->>LLM: generate([SystemMessage, HumanMessage])
    LLM->>DIAL: POST /chat/completions
    DIAL-->>LLM: completion
    LLM-->>RAG: LLMResult
    RAG-->>Main: answer text
    
    Main->>User: Display answer
```

### Vector Index Creation Flow

```mermaid
flowchart TD
    A[Start] --> B{Index exists?}
    B -->|Yes| C[Load from disk]
    B -->|No| D[Read microwave_manual.txt]
    
    D --> E[TextLoader.load]
    E --> F[RecursiveCharacterTextSplitter]
    F --> G[Split into chunks<br/>size=300, overlap=50]
    
    G --> H[Embed all chunks]
    H --> I[AzureOpenAIEmbeddings.embed_documents]
    I --> J[FAISS.from_documents]
    
    J --> K[Create FAISS index]
    K --> L[Save to microwave_faiss_index/]
    
    L --> M[Return VectorStore]
    C --> M
    M --> N[End]
    
    style B fill:#fff4e1
    style D fill:#e1f5ff
    style H fill:#ffe1f5
    style L fill:#e1ffe1
```

## Module Breakdown

### `task/app.py`

**Lines of Code**: 173  
**Entry Point**: `main()` function (line 151)  
**Key Classes**: `MicrowaveRAG`

**Module Responsibilities**:
- RAG pipeline orchestration
- Vector store initialization and persistence
- Document chunking and embedding
- Similarity search and context retrieval
- Prompt augmentation
- LLM answer generation

**External Dependencies**:
- `langchain_community`: Document loaders, vector stores
- `langchain_text_splitters`: Text chunking
- `langchain_openai`: Azure OpenAI integrations
- `langchain_core`: Message types, abstractions

### `task/_constants.py`

**Lines of Code**: 4  
**Exports**: `DIAL_URL`, `API_KEY`

**Module Responsibilities**:
- Centralized configuration
- Environment variable management

### `task/microwave_manual.txt`

**Data File**: Knowledge base  
**Format**: UTF-8 plaintext  
**Content**: Microwave model DW 395 HCG manual

## Design Decisions

### Why FAISS for Vector Storage?

**Decision**: Use FAISS for local vector similarity search  
**Rationale**:
- Educational focus: Students learn vector indexing locally
- No external database dependencies
- Fast similarity search (<10ms for 4 results)
- Persistent index avoids re-embedding on every run

**Trade-offs**:
- Not horizontally scalable
- In-memory constraint (~1GB for large knowledge bases)
- No multi-user support

See [ADR-001: FAISS for Vector Storage](./adr/ADR-001-faiss-vector-storage.md)

### Why RecursiveCharacterTextSplitter?

**Decision**: Use `RecursiveCharacterTextSplitter` with separators `["\n\n", "\n", "."]`  
**Parameters**: `chunk_size=300`, `chunk_overlap=50`

**Rationale**:
- Preserves semantic coherence (splits on paragraphs, then sentences)
- `chunk_size=300` balances context vs. specificity
- `chunk_overlap=50` prevents context loss at boundaries

**Trade-offs**:
- May split mid-sentence if paragraph > 300 chars
- Overlap increases storage by ~17%

See [ADR-003: Document Chunking Strategy](./adr/ADR-003-chunking-strategy.md)

### Why Three Separate Methods?

**Decision**: `retrieve_context`, `augment_prompt`, `generate_answer` as separate methods

**Rationale**:
- Educational clarity: Students see each RAG stage explicitly
- Debugging: Inspect intermediate outputs
- Flexibility: Tune each stage independently

**Trade-offs**:
- More verbose than a single `query()` method
- Requires manual orchestration in `main()`

## Integration Points

### 1. DIAL API Integration

**Service**: EPAM's AI Proxy for Azure OpenAI  
**Endpoint**: `https://ai-proxy.lab.epam.com`  
**Authentication**: Bearer token (`DIAL_API_KEY`)

**Used For**:
- Text embeddings (`text-embedding-3-small-1`)
- Chat completions (`gpt-4o`)

**Network Requirements**:
- EPAM VPN connection required
- HTTPS (TLS 1.2+)

**Error Handling**: TODO: Currently none. See [Roadmap](./roadmap.md) for planned improvements.

### 2. Filesystem Integration

**Index Persistence**:
- **Path**: `./microwave_faiss_index/`
- **Format**: Binary FAISS index + pickle metadata
- **Safety**: `allow_dangerous_deserialization=True` (trusted source)

**Knowledge Base**:
- **Path**: `task/microwave_manual.txt` (relative to script location)
- **Encoding**: UTF-8
- **Error Handling**: Raises `FileNotFoundError` if missing

## Constraints & Trade-offs

### Performance Constraints

| Operation | Latency | Bottleneck |
|-----------|---------|------------|
| Index creation | ~30s | DIAL API embeddings |
| Query embedding | ~200ms | DIAL API network |
| Similarity search | <10ms | FAISS in-memory |
| LLM generation | ~2-5s | DIAL API (Azure OpenAI) |

**Total Query Time**: ~2.5-5.5s (dominated by LLM generation)

### Scalability Constraints

- **Single-threaded**: No concurrent query support
- **In-memory index**: Limited by RAM (~1GB for 10K chunks)
- **Local storage**: No distributed deployment

### Cost Constraints

**DIAL API Usage**:
- Embeddings: ~$0.0001 per 1K tokens
- Chat completions: ~$0.03 per 1K tokens (GPT-4o)

**Estimated Cost per Query**:
- Retrieval: ~$0.0001 (400 tokens embedded)
- Generation: ~$0.003 (100 tokens avg response)
- **Total**: ~$0.0031 per query

### Security Constraints

- **API Key**: Stored in environment variable (not version-controlled)
- **Network**: Requires EPAM VPN (internal proxy)
- **Data**: Knowledge base is public (microwave manual)

**Known Vulnerabilities**:
- `allow_dangerous_deserialization=True` in FAISS loading (acceptable for trusted local index)

## Open Questions

- **Monitoring**: How to instrument for production? (metrics, logging, tracing)
- **Error handling**: Should failures retry? Circuit breaker pattern?
- **Multi-tenancy**: How to support multiple users or knowledge bases?
- **Performance**: Can we batch queries or use async I/O?

See [Roadmap](./roadmap.md) for planned explorations.

---

**Related Documentation**:
- [API Reference](./api.md) - Detailed method signatures
- [ADR Index](./adr/README.md) - Architectural decision records
- [Testing Guide](./testing.md) - Validation strategies
