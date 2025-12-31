---
title: Roadmap
description: Future enhancements, learning milestones, and planned improvements
version: 1.0.0
last_updated: 2025-12-30
related: [README.md, architecture.md, changelog.md]
tags: [roadmap, future, enhancements, backlog]
---

# Roadmap

## Overview

This roadmap outlines planned enhancements for the AI DIAL RAG Overview project. Items are categorized by priority and effort level.

**Project Phase**: Educational/Learning (v1.0)  
**Target Audience**: Python developers learning RAG concepts

## Roadmap Categories

- 🎓 **Educational**: Improves learning experience
- 🛠️ **Technical**: Enhances functionality
- 🔒 **Security**: Improves safety and compliance
- 📊 **Observability**: Adds monitoring and debugging
- ⚡ **Performance**: Optimizes speed or resources

---

## Short Term (1-3 months)

### 🎓 Add Interactive Tutorial Mode
**Priority**: High  
**Effort**: Medium (8-12 hours)

**Description**: Step-by-step guided tutorial explaining each RAG stage with examples.

**Implementation**:
```python
# task/tutorial.py
def tutorial_mode():
    print("=== RAG Tutorial: Stage 1 - Retrieval ===")
    print("Let's see how we find relevant chunks...")
    # Interactive walkthrough with pauses
```

**Success Criteria**:
- Guided walkthrough of 3 RAG stages
- Example queries with explanations
- Visualization of retrieved chunks

---

### 🛠️ Add Error Handling & Retries
**Priority**: High  
**Effort**: Low (4-6 hours)

**Description**: Graceful handling of DIAL API errors (network, rate limits, timeouts).

**Current State**: All API errors unhandled (crashes)

**Implementation**:
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=10))
def call_dial_api(prompt):
    # API call with automatic retries
```

**Error Scenarios**:
- 401 Unauthorized → Clear error message with setup instructions
- 429 Rate Limit → Exponential backoff retry
- Network timeout → Retry with jitter
- 500 Server Error → Graceful degradation

**Related**: [Setup: Troubleshooting](./setup.md#troubleshooting)

---

### 📊 Add Logging & Instrumentation
**Priority**: Medium  
**Effort**: Medium (6-8 hours)

**Description**: Structured logging for debugging and performance analysis.

**Implementation**:
```python
import logging
import structlog

log = structlog.get_logger()

def retrieve_context(self, query):
    log.info("retrieval_started", query=query, k=4)
    # ... retrieval logic
    log.info("retrieval_completed", chunks_found=len(results), latency_ms=latency)
```

**Logged Events**:
- Query received
- Retrieval: embeddings call, similarity search, chunks found
- Augmentation: prompt length
- Generation: LLM call, tokens used, latency

**Output Format**: JSON for parsing and analysis

---

### 🔒 Add .env File Support
**Priority**: Medium  
**Effort**: Low (1-2 hours)

**Description**: Use `python-dotenv` for persistent configuration.

**Implementation**:
```python
# task/_constants.py
from dotenv import load_dotenv
load_dotenv()
```

```bash
# .env (user creates locally, in .gitignore)
DIAL_API_KEY=your-key-here
```

**Benefits**: No need to export environment variable in every shell session

**Related**: [ADR-002: Environment Configuration](./adr/ADR-002-environment-config.md)

---

## Medium Term (3-6 months)

### 🛠️ Add Automated Tests
**Priority**: High  
**Effort**: High (16-24 hours)

**Description**: Unit and integration tests for reliability.

**Coverage Goals**:
- Unit tests: `retrieve_context`, `augment_prompt`, `generate_answer`
- Integration tests: Full pipeline with mock LLM
- Regression tests: Sample queries with expected outputs

**Framework**: `pytest` with fixtures

**Implementation**:
```python
# tests/test_rag.py
import pytest
from task.app import MicrowaveRAG

@pytest.fixture
def rag_instance():
    return MicrowaveRAG(mock_embeddings, mock_llm)

def test_retrieve_context_returns_string(rag_instance):
    context = rag_instance.retrieve_context("test")
    assert isinstance(context, str)
```

**CI Integration**: GitHub Actions for automated test runs

**Related**: [Testing Guide](./testing.md)

---

### ⚡ Add Caching Layer
**Priority**: Medium  
**Effort**: Medium (8-12 hours)

**Description**: Cache query embeddings and LLM responses for faster repeated queries.

**Implementation**:
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def embed_query_cached(query: str):
    return self.embeddings.embed_query(query)
```

**Cache Strategy**:
- **Embeddings cache**: LRU cache (128 queries)
- **Response cache**: Redis or file-based (for demo purposes)

**Benefits**:
- Retrieval: 200ms → 10ms for cached queries
- Generation: 3s → instant for repeated queries

**Trade-offs**:
- Memory usage increase
- Stale data if manual changes

---

### 🎓 Add Multiple Knowledge Bases
**Priority**: Medium  
**Effort**: High (16-20 hours)

**Description**: Support for multiple manuals (different appliances).

**Implementation**:
```python
class MultiDocumentRAG:
    def __init__(self, knowledge_bases: dict):
        self.vectorstores = {
            name: self._load_vectorstore(path)
            for name, path in knowledge_bases.items()
        }
    
    def retrieve_context(self, query, knowledge_base="microwave"):
        return self.vectorstores[knowledge_base].similarity_search(query)
```

**Use Case**: Students can compare RAG across different domains

---

### 📊 Add Web UI (Gradio/Streamlit)
**Priority**: Medium  
**Effort**: Medium (10-16 hours)

**Description**: Simple web interface for easier interaction.

**Framework**: Gradio (simpler) or Streamlit (richer)

**Features**:
- Query input box
- Retrieval visualization (scores, chunks)
- Answer display with sources
- Parameter sliders (k, score threshold)

**Implementation**:
```python
import gradio as gr

def query_interface(user_query, k, score):
    context = rag.retrieve_context(user_query, k=k, score=score)
    # ... rest of pipeline
    return answer, context

gr.Interface(
    fn=query_interface,
    inputs=[
        gr.Textbox(label="Question"),
        gr.Slider(1, 10, value=4, label="Chunks (k)"),
        gr.Slider(0, 1, value=0.3, label="Score Threshold")
    ],
    outputs=["text", "text"]
).launch()
```

---

## Long Term (6-12 months)

### 🛠️ Add Conversation History
**Priority**: Low  
**Effort**: High (20-30 hours)

**Description**: Multi-turn conversation with context retention.

**Implementation**:
- Session management (in-memory or Redis)
- Conversation history in prompts
- Follow-up question understanding

**Challenges**:
- Context window management (token limits)
- Relevance decay over turns

---

### ⚡ Migrate to Async Pipeline
**Priority**: Low  
**Effort**: High (24-32 hours)

**Description**: Asynchronous I/O for concurrent queries and faster processing.

**Implementation**:
```python
async def retrieve_context_async(self, query):
    embedding = await self.embeddings.aembed_query(query)
    results = await asyncio.to_thread(self.vectorstore.similarity_search, embedding)
    return results
```

**Benefits**:
- Concurrent query processing
- Non-blocking I/O for API calls
- Better resource utilization

**Trade-offs**:
- Increased complexity
- Debugging difficulty

---

### 🎓 Add Evaluation Framework
**Priority**: Medium  
**Effort**: High (16-24 hours)

**Description**: Automated RAG quality evaluation metrics.

**Metrics**:
- **Retrieval**: Precision@K, Recall@K, MRR
- **Generation**: BLEU, ROUGE, BERTScore
- **End-to-End**: Faithfulness, answer relevance

**Implementation**:
```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevance

results = evaluate(
    dataset=test_queries,
    metrics=[faithfulness, answer_relevance]
)
```

**Use Case**: Compare different chunking strategies or models

---

### 🔒 Add Rate Limiting & Quotas
**Priority**: Low  
**Effort**: Medium (8-12 hours)

**Description**: Protect DIAL API from excessive usage.

**Implementation**:
```python
from ratelimit import limits, sleep_and_retry

@sleep_and_retry
@limits(calls=10, period=60)  # 10 calls per minute
def call_llm(prompt):
    return self.llm_client.generate(prompt)
```

---

## Research & Exploration

### 🛠️ Advanced Retrieval Techniques
**Status**: Research phase

**Ideas**:
- **Hybrid search**: Combine keyword (BM25) + vector search
- **Reranking**: Use cross-encoder to reorder retrieved chunks
- **Query expansion**: Generate multiple query variations
- **Metadata filtering**: Filter by section, timestamp, etc.

**References**:
- [LangChain: Hybrid Search](https://python.langchain.com/docs/modules/data_connection/retrievers/hybrid)
- [Cohere Rerank](https://docs.cohere.com/docs/reranking)

---

### ⚡ Alternative Embedding Models
**Status**: Research phase

**Options**:
- **Smaller models**: `text-embedding-ada-002` (cheaper)
- **Larger models**: `text-embedding-3-large` (better quality)
- **Open-source**: `sentence-transformers/all-MiniLM-L6-v2`

**Trade-offs**:
| Model | Dimensions | Cost/1K | Quality |
|-------|-----------|---------|---------|
| ada-002 | 1536 | $0.0001 | Good |
| 3-small | 1536 | $0.0001 | Better |
| 3-large | 3072 | $0.0003 | Best |

---

### 🎓 RAG Variations to Demonstrate
**Status**: Idea backlog

**Variations**:
- **Self-querying RAG**: LLM generates search metadata
- **Hypothetical Document Embeddings (HyDE)**: Generate fake document, embed, search
- **Parent Document Retrieval**: Retrieve small chunks, return larger parent sections
- **Multi-query RAG**: Generate multiple queries from user input

**Educational Value**: Shows students different RAG patterns

---

## Backlog & Ideas

### Lower Priority Enhancements

- [ ] Export conversation history (JSON, markdown)
- [ ] Add dark mode to CLI output
- [ ] Multilingual support (Spanish, French manuals)
- [ ] Voice input/output (speech-to-text, TTS)
- [ ] Integration with Slack/Teams for chatbot
- [ ] Mobile app (React Native wrapper)
- [ ] Docker containerization for easy deployment
- [ ] Kubernetes deployment example
- [ ] Monitoring dashboard (Grafana)
- [ ] Cost tracking and optimization

---

## Completed Milestones

### Version 1.0 (December 2025)
- [x] Basic RAG pipeline (Retrieval, Augmentation, Generation)
- [x] FAISS vector storage
- [x] CLI interface
- [x] Sample microwave manual knowledge base
- [x] Comprehensive documentation
- [x] Architecture decision records

---

## How to Contribute

1. **Pick an item**: Choose from Short/Medium Term sections
2. **Create issue**: Describe approach and timeline
3. **Branch & develop**: Use feature branches
4. **Add tests**: All new features require tests
5. **Update docs**: Documentation for new features
6. **Submit PR**: Include ADR if architectural change

---

## Related Documentation

- [Architecture](./architecture.md) - Current system design
- [ADR Index](./adr/README.md) - Decision records
- [Changelog](./changelog.md) - Historical changes
- [Testing Guide](./testing.md) - Quality assurance

---

**Maintained by**: EPAM AI/ML Team  
**Last Review**: 2025-12-30  
**Next Review**: 2026-03-30
