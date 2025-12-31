# ADR-003: Document Chunking Strategy

## Status
**Accepted** (2025-12-30)

## Context

The RAG system must process the microwave manual (497 lines, ~25KB) into vector embeddings for similarity search. This requires splitting the large document into smaller chunks that balance:
- **Semantic coherence**: Chunks must be meaningful units (not random fragments)
- **Search relevance**: Retrieved chunks should contain complete answers without excessive padding
- **Embedding efficiency**: Smaller chunks = faster embedding calls and better search precision
- **Overlap handling**: Prevent losing information at chunk boundaries

### Requirements
- Chunks must be small enough for targeted retrieval (avoid retrieving entire manual)
- Chunks must be large enough to contain complete information (single sentence insufficient)
- Chunking strategy must be deterministic and reproducible
- Should support easy re-chunking if strategy changes

### Constraints
- Limited to document structure available in plain text (no markdown/HTML structure)
- Must use LangChain's `RecursiveCharacterTextSplitter` (compatibility requirement)
- One-time operation at application startup (performance not critical)

## Decision

**Use `RecursiveCharacterTextSplitter` with:**
- **Chunk size**: 300 characters
- **Overlap**: 50 characters
- **Separators**: `["\n\n", "\n", "."]` (recursive, preferring larger units)

### Implementation

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=300,
    chunk_overlap=50,
    separators=["\n\n", "\n", "."]
)
chunks = text_splitter.split_documents(documents)
```

### Rationale

**300 characters**:
- Typical paragraph: 200-400 characters
- One complete sentence to few sentences per chunk
- Results in ~100-150 chunks from 25KB manual (manageable for FAISS)

**50 characters overlap**:
- Prevents losing information at boundaries
- Ensures context continuity (typically ~1-2 words carried forward)
- Minimal redundancy (50/300 = 16.7% overlap ratio)

**Separator hierarchy** `["\n\n", "\n", "."]`:
- **Primary**: Double newline (paragraph boundary) - highest priority
- **Secondary**: Single newline (sentence/list item boundary)
- **Fallback**: Period (sentence boundary) - if no structure present
- Ensures chunks respect document structure before resorting to hard splitting

## Consequences

### Positive
- **Semantic preservation**: Recursive separator strategy maintains document structure
- **Reasonable overlap**: Prevents information loss while minimizing duplication
- **LangChain native**: No custom splitting logic needed
- **Deterministic**: Same input always produces same chunks
- **Flexible**: Easy to tune by changing parameters
- **Observable**: Students can inspect chunks and understand retrieval behavior

### Negative
- **Static parameters**: Fixed chunk size may not work for all document types
- **No semantic awareness**: Splits on structure, not meaning (may split cohesive topics)
- **Boundary artifacts**: Overlap creates duplicate content in vector store
- **Limited precision**: May include unnecessary context in edge cases

### Neutral
- **No metadata**: Chunks don't preserve source section information (acceptable for small manual)
- **Character-based**: Splits on characters, not tokens (could differ from embedding tokenization)

## Alternatives Considered

### 1. Semantic Chunking (Custom Implementation)

**Approach**: Split based on semantic similarity between sentences

```python
# Pseudo-code: split when semantic distance > threshold
chunks = []
sentences = document.split('. ')
current_chunk = [sentences[0]]
for sentence in sentences[1:]:
    if semantic_distance(current_chunk[-1], sentence) > 0.5:
        chunks.append('. '.join(current_chunk))
        current_chunk = [sentence]
    else:
        current_chunk.append(sentence)
```

**Pros**:
- Respects semantic boundaries (maximizes information coherence)
- Adapts to document content, not just structure
- Potentially higher retrieval quality

**Cons**:
- **Requires embedding model**: Need to call embedding API during chunking (~30s overhead)
- **Complex implementation**: Beyond basic text splitting
- **Overkill for small document**: Not needed for 25KB manual
- **Educational overhead**: Distracts from RAG pipeline learning

**Verdict**: Rejected (unnecessary complexity, time overhead)

---

### 2. Token-Based Chunking

**Approach**: Split by token count (respecting embedding model's tokenization)

```python
# Using tiktoken or similar
tokenizer = tiktoken.encoding_for_model("gpt-4")
token_size = 100  # ~75 characters per token (rough estimate)
chunks = []
current_tokens = []
for token in tokenizer.encode(document):
    current_tokens.append(token)
    if len(current_tokens) >= token_size:
        chunks.append(tokenizer.decode(current_tokens))
        current_tokens = current_tokens[-10:]  # overlap
```

**Pros**:
- Aligns with embedding model's actual tokenization
- More consistent with how model processes text
- Avoids splitting mid-token

**Cons**:
- **Extra dependency**: Requires `tiktoken` package
- **Model-specific**: Different models have different tokenizers
- **Complexity**: More difficult to reason about for students
- **Not in LangChain core**: Requires custom implementation

**Verdict**: Rejected (unnecessary dependency, complexity)

---

### 3. Fixed-Size Tokens (Simple Alternative)

**Approach**: Split by word count (simpler than tokens)

```python
words = document.split()
chunk_size = 50  # words per chunk
chunks = [' '.join(words[i:i+chunk_size]) 
          for i in range(0, len(words), chunk_size-10)]  # 10-word overlap
```

**Pros**:
- Very simple to understand
- No dependencies
- Deterministic

**Cons**:
- **Ignores document structure**: May split paragraphs awkwardly
- **Poor semantic boundaries**: Word count doesn't align with meaning
- **Not LangChain standard**: Deviates from framework practices
- **Less configurable**: Can't specify separators

**Verdict**: Rejected (poor semantic boundaries)

---

### 4. Manual Section-Based Splitting

**Approach**: Hardcode splits based on manual's known sections

```python
# Split by manual structure:
# - Safety, Operation, Cleaning, Specifications, etc.
sections = {
    'safety': document[0:1000],
    'operation': document[1000:5000],
    ...
}
```

**Pros**:
- Perfect semantic alignment (respects domain structure)
- Maximum relevance for queries

**Cons**:
- **Not scalable**: Breaks if manual content changes
- **Not generalizable**: Students can't apply to other documents
- **Maintenance burden**: Must update if manual structure changes
- **Anti-learning**: Defeats purpose of learning chunking strategies

**Verdict**: Rejected (not generalizable, poor educational value)

---

## Trade-offs Analysis

| Criterion | Recursive-Char | Semantic | Token-Based | Word-Based | Section-Based |
|-----------|---|---|---|---|---|
| **Implementation Simplicity** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Semantic Quality** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Educational Value** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Performance (Chunking)** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Scalability** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| **Generalizability** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ |

**Winner for this use case**: Recursive Character Splitting (best balance of simplicity, quality, and educational value)

---

## Implementation Notes

### Empirical Results

For the microwave manual:
- **Total chunks created**: ~100-120 chunks
- **Average chunk size**: ~250 characters (goal: 300, adjusted by boundaries)
- **Overlap ratio**: ~16.7% (50/300)
- **Processing time**: <100ms

### Parameter Tuning Guide

**If retrieval returns too much context**:
- Reduce `chunk_size` to 200-250 characters
- Reduces overlap to minimize duplication

**If retrieval misses relevant information**:
- Increase `chunk_overlap` to 75-100 characters
- Increase `chunk_size` to 400-500 characters

**If chunks break semantic units**:
- Add more separators (e.g., ": " for list markers)
- Increase `chunk_size` to preserve paragraphs

### Re-chunking Strategy

When to rebuild the index:
1. **Manual content changes**: Delete `microwave_faiss_index/` directory
2. **Parameters change**: Same as above
3. **Embedding model changes**: Index becomes incompatible

**How to rebuild**:
```bash
rm -rf microwave_faiss_index/
python -m task.app  # Creates new index on startup
```

### Character vs. Token Consideration

Note: Chunk size is specified in **characters**, not tokens. Approximate conversion:
- 1 token ≈ 3-4 characters (average)
- 300 characters ≈ 75-100 tokens
- Azure's `text-embedding-3-small-1` model can handle up to 8192 tokens

Our 300-character chunks are well within model limits.

---

## Related Decisions

- [ADR-001: FAISS Vector Storage](./ADR-001-faiss-vector-storage.md) - How chunks are indexed
- [ADR-004: Synchronous Pipeline](./ADR-004-synchronous-pipeline.md) - How chunking integrates into pipeline
