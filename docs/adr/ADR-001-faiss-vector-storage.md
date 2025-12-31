# ADR-001: FAISS for Vector Storage

## Status
**Accepted** (2025-12-30)

## Context

The RAG system requires a vector database to:
1. Store embeddings of document chunks from the microwave manual
2. Perform fast similarity search to retrieve relevant chunks for user queries
3. Persist the index to avoid re-embedding on every application run

### Requirements
- **Performance**: Similarity search must complete in <100ms for 4-8 results
- **Simplicity**: Educational project—students should understand the technology easily
- **Cost**: Minimal infrastructure/operational costs
- **Deployment**: Single-machine, local development environment
- **Scale**: Small knowledge base (~500 chunks, ~750KB embeddings)

### Constraints
- Python ecosystem (LangChain integration)
- No budget for cloud vector database services
- Students should run locally without external dependencies
- No multi-user or distributed deployment requirements

## Decision

**Use FAISS (Facebook AI Similarity Search) with local filesystem persistence.**

### Configuration
- **Index Type**: Flat (brute-force) index for exact search
- **Distance Metric**: L2 (Euclidean distance, FAISS default)
- **Storage**: Local filesystem (`./microwave_faiss_index/`)
- **Serialization**: Pickle + binary index file

### Implementation
```python
# Create index
vectorstore = FAISS.from_documents(chunks, embeddings)
vectorstore.save_local("microwave_faiss_index")

# Load index
vectorstore = FAISS.load_local(
    folder_path="microwave_faiss_index",
    embeddings=embeddings,
    allow_dangerous_deserialization=True
)
```

## Consequences

### Positive
- **Zero infrastructure cost**: No database server or cloud service fees
- **Fast similarity search**: <10ms for 4 results (in-memory flat index)
- **Simple setup**: Works out-of-the-box with `pip install faiss-cpu`
- **LangChain integration**: Native `VectorStore` interface compatibility
- **Persistence**: Avoid re-embedding (~30s) on every run
- **Deterministic**: Exact search (no approximation errors)
- **Educational clarity**: Students can inspect index files and understand storage

### Negative
- **Not scalable**: Limited by single-machine RAM (~1GB for 10K chunks)
- **No concurrency**: File-based locking, no multi-user support
- **No production features**: No replication, monitoring, or access control
- **Security warning**: `allow_dangerous_deserialization=True` required for pickle loading

### Neutral
- **Local-only**: Can't share index across machines (acceptable for learning project)
- **Manual versioning**: Must rebuild index if chunking strategy changes

## Alternatives Considered

### 1. Chroma (Embedded Vector Database)

**Pros**:
- More production-ready (metadata filtering, updates, deletes)
- Built-in persistence without manual save/load
- Better multi-collection support

**Cons**:
- Additional dependency (`chromadb` package)
- More complex API for simple use case
- Overkill for single-document, read-only scenario

**Verdict**: Rejected (unnecessary complexity)

---

### 2. Pinecone (Cloud Vector Database)

**Pros**:
- Horizontally scalable
- Managed service (no infrastructure)
- Production-grade performance and reliability

**Cons**:
- **Cost**: $70/month minimum for hosted service
- **Network dependency**: Requires internet for every query
- **Educational gap**: Students don't learn vector search internals
- **Setup friction**: Account creation, API keys

**Verdict**: Rejected (cost + educational goals)

---

### 3. Elasticsearch with Dense Vectors

**Pros**:
- Production-proven search infrastructure
- Rich query language and filtering
- Mature ecosystem and tooling

**Cons**:
- **Heavy dependency**: Requires Java, ~2GB RAM for Elasticsearch server
- **Setup complexity**: Too much operational overhead for learning project
- **Overkill**: Designed for full-text search + vectors, we only need vectors

**Verdict**: Rejected (complexity + resource requirements)

---

### 4. Qdrant (Open-Source Vector Database)

**Pros**:
- Modern Rust-based vector database
- Easy Docker deployment
- Rich filtering and metadata support

**Cons**:
- **External dependency**: Requires Docker or binary installation
- **Network overhead**: Client-server architecture (localhost latency)
- **Setup friction**: More moving parts than FAISS

**Verdict**: Rejected (unnecessary external process)

---

### 5. NumPy + Manual Search (No Library)

**Pros**:
- Maximum educational value (students implement similarity search)
- Zero dependencies beyond NumPy
- Full control over algorithm

**Cons**:
- **Time-consuming**: Building index management from scratch
- **Reinventing wheel**: FAISS is well-tested and optimized
- **Scope creep**: Shifts focus from RAG concepts to vector math

**Verdict**: Rejected (not core learning objective)

---

## Trade-offs Analysis

| Criterion | FAISS | Chroma | Pinecone | Elasticsearch |
|-----------|-------|--------|----------|---------------|
| **Setup Time** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐ |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Scalability** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Cost** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐ |
| **Educational Value** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ |

**Winner for this use case**: FAISS (best balance for educational, small-scale project)

---

## Implementation Notes

### Security Consideration
`allow_dangerous_deserialization=True` is required because FAISS uses pickle for metadata serialization. This is **safe** in our context because:
- Index files are generated locally (trusted source)
- No user-uploaded or network-fetched indexes
- Single-user development environment

**Production systems** should use secure serialization or signed index files.

### Performance Characteristics

**Index Creation**:
- Time: ~30 seconds (dominated by DIAL API embedding calls)
- Disk Usage: ~150KB (index.faiss + index.pkl)

**Index Loading**:
- Time: ~2 seconds (file I/O + deserialization)
- Memory: ~2MB (1536-dim vectors × 100 chunks)

**Similarity Search**:
- Time: <10ms for k=4 (flat index, brute-force)
- Scales linearly: O(n×d) where n=chunks, d=dimensions

### When to Rebuild Index
- Knowledge base content changes
- Chunking parameters modified (`chunk_size`, `chunk_overlap`)
- Embedding model changed

**How**: Delete `microwave_faiss_index/` directory and restart application.

---

## Future Considerations

If the project evolves beyond educational scope:

1. **For multi-document RAG**: Consider Chroma (better collection management)
2. **For production deployment**: Evaluate Pinecone or Qdrant (managed, scalable)
3. **For large knowledge bases**: Implement approximate search (IVF or HNSW indexes)
4. **For multi-user access**: Add database layer or Redis-backed vector store

---

## References

- [FAISS GitHub](https://github.com/facebookresearch/faiss)
- [LangChain FAISS Integration](https://python.langchain.com/docs/integrations/vectorstores/faiss)
- [Vector Database Comparison (2024)](https://www.datastax.com/guides/vector-database-comparison)

---

**Decision Made By**: Architecture team  
**Date**: 2025-12-30  
**Supersedes**: None  
**Related ADRs**: [ADR-003 (Chunking Strategy)](./ADR-003-chunking-strategy.md)
