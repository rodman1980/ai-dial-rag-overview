---
title: API Reference
description: Public interfaces, classes, methods, and usage examples for the RAG system
version: 1.0.0
last_updated: 2025-12-30
related: [architecture.md, README.md]
tags: [api, reference, classes, methods]
---

# API Reference

## Table of Contents
- [Overview](#overview)
- [MicrowaveRAG Class](#microwaverag-class)
- [Configuration Constants](#configuration-constants)
- [Prompt Templates](#prompt-templates)
- [Usage Examples](#usage-examples)
- [Integration Patterns](#integration-patterns)

## Overview

This document describes the public API for the AI DIAL RAG Overview system. The primary interface is the `MicrowaveRAG` class, which orchestrates document retrieval, prompt augmentation, and answer generation.

**Module**: `task.app`  
**Entry Point**: `main()` function  
**Key Dependency**: `langchain_openai`, `langchain_community`, `faiss-cpu`

## MicrowaveRAG Class

### Class Definition

```python
class MicrowaveRAG:
    """
    RAG system for microwave manual question-answering.
    
    Implements a three-stage pipeline:
    1. Retrieval: Find relevant document chunks via FAISS similarity search
    2. Augmentation: Format query with retrieved context
    3. Generation: Generate answer using Azure OpenAI
    
    Attributes:
        llm_client (AzureChatOpenAI): Chat completion client
        embeddings (AzureOpenAIEmbeddings): Text embedding client
        vectorstore (VectorStore): FAISS vector store for similarity search
    """
```

### Constructor

#### `__init__(embeddings, llm_client)`

Initialize the RAG system with embedding and LLM clients.

**Parameters**:
- `embeddings` (`AzureOpenAIEmbeddings`): Azure OpenAI embeddings client for text vectorization
- `llm_client` (`AzureChatOpenAI`): Azure OpenAI chat client for answer generation

**Returns**: `MicrowaveRAG` instance

**Side Effects**:
- Loads or creates FAISS vector index
- Reads `microwave_manual.txt` if index doesn't exist
- Creates `microwave_faiss_index/` directory on first run

**Example**:
```python
from langchain_openai import AzureOpenAIEmbeddings, AzureChatOpenAI
from pydantic import SecretStr
from task._constants import DIAL_URL, API_KEY

embeddings = AzureOpenAIEmbeddings(
    deployment="text-embedding-3-small-1",
    azure_endpoint=DIAL_URL,
    api_key=SecretStr(API_KEY)
)

llm_client = AzureChatOpenAI(
    temperature=0.0,
    azure_deployment="gpt-4o",
    azure_endpoint=DIAL_URL,
    api_key=SecretStr(API_KEY),
    api_version=""
)

rag = MicrowaveRAG(embeddings, llm_client)
```

**Time Complexity**:
- First run (no index): O(n) where n = number of document chunks (~30s)
- Subsequent runs: O(1) (~2s to load index)

**Exceptions**:
- `FileNotFoundError`: If `microwave_manual.txt` missing
- `ValueError`: If embeddings/llm_client invalid
- Network errors from DIAL API (unhandled)

---

### Public Methods

#### `retrieve_context(query, k=4, score=0.3)`

Retrieve relevant document chunks for a given query.

**Stage**: 1 (Retrieval)

**Parameters**:
- `query` (`str`): User's question or search query
- `k` (`int`, optional): Maximum number of chunks to retrieve. Default: 4
- `score` (`float`, optional): Minimum similarity score threshold (0.0-1.0). Default: 0.3

**Returns**: `str`
- Concatenated text of all retrieved chunks, separated by `\n\n`
- Empty string if no chunks meet the threshold

**Example**:
```python
query = "How do I clean the microwave?"
context = rag.retrieve_context(query, k=4, score=0.3)

print(f"Retrieved {len(context)} characters of context")
# Output: Retrieved 1234 characters of context
```

**Output Format**:
```
Chunk 1 text here...

Chunk 2 text here...

Chunk 3 text here...
```

**Algorithm**:
1. Embed query using `AzureOpenAIEmbeddings`
2. Perform FAISS similarity search with relevance scores
3. Filter chunks with `score >= threshold`
4. Return top `k` chunks concatenated

**Performance**:
- Embedding: ~200ms (DIAL API)
- Similarity search: <10ms (FAISS in-memory)
- **Total**: ~210ms

**Console Output** (verbose):
```
====================================================================================================
🔍 STEP 1: RETRIEVAL
----------------------------------------------------------------------------------------------------
Query: 'How do I clean the microwave?'
Searching for top 4 most relevant chunks with similarity score 0.3:
Score: 0.78
Content: To clean the glass tray...
Score: 0.65
Content: Use a damp cloth...
====================================================================================================
```

**Parameter Tuning**:
- `k=2`: Fast, less context (for simple queries)
- `k=6`: More context, slower (for complex queries)
- `score=0.2`: More permissive (broader context)
- `score=0.5`: Stricter (only highly relevant chunks)

---

#### `augment_prompt(query, context)`

Format user query with retrieved context into a structured prompt.

**Stage**: 2 (Augmentation)

**Parameters**:
- `query` (`str`): Original user question
- `context` (`str`): Retrieved context from `retrieve_context()`

**Returns**: `str`
- Formatted prompt ready for LLM generation

**Example**:
```python
query = "How do I clean the microwave?"
context = "To clean the glass tray, wash with mild soap..."

augmented_prompt = rag.augment_prompt(query, context)
print(augmented_prompt)
```

**Output**:
```
Context:
To clean the glass tray, wash with mild soap...

Question:
How do I clean the microwave?
```

**Template Structure**:
```python
USER_PROMPT = """
Context:
{context}

Question:
{query}
"""
```

**Design Note**: Simple template for educational clarity. Production systems might use more sophisticated prompt engineering (e.g., few-shot examples, chain-of-thought).

**Performance**: <1ms (string formatting)

**Console Output** (verbose):
```
🔗 STEP 2: AUGMENTATION
----------------------------------------------------------------------------------------------------
Context:
To clean the glass tray...

Question:
How do I clean the microwave?
====================================================================================================
```

---

#### `generate_answer(augmented_prompt)`

Generate answer using Azure OpenAI given an augmented prompt.

**Stage**: 3 (Generation)

**Parameters**:
- `augmented_prompt` (`str`): Formatted prompt from `augment_prompt()`

**Returns**: `str`
- Generated answer text from LLM

**Example**:
```python
augmented_prompt = "Context: ...\n\nQuestion: How do I clean the microwave?"
answer = rag.generate_answer(augmented_prompt)

print(answer)
# Output: "To clean your microwave, first remove the glass tray..."
```

**Message Structure**:
```python
messages = [
    SystemMessage(content=SYSTEM_PROMPT),  # Role definition
    HumanMessage(content=augmented_prompt)  # User query + context
]
```

**System Prompt**:
```
You are a helpful assistant.
```

**Note**: The actual system prompt in `app.py` is more detailed, instructing the LLM to:
- Answer only from provided context
- Cite sources
- State when information is unavailable

**LLM Configuration**:
- Model: `gpt-4o` (Azure OpenAI)
- Temperature: `0.0` (deterministic)
- Max tokens: Default (~800)

**Performance**:
- LLM generation: 2-5s (depends on response length)
- Network latency: ~100ms

**Console Output** (verbose):
```
🤖 STEP 3: GENERATION
----------------------------------------------------------------------------------------------------
Response: To clean your microwave, first remove the glass tray and wash it with mild soap and warm water...
```

**Exceptions**:
- Network errors (DIAL API timeout)
- Rate limiting (429 errors)
- Invalid API key (401 errors)

*Currently unhandled—see [Roadmap](./roadmap.md) for planned error handling.*

---

### Private Methods

#### `_setup_vectorstore()`

Initialize vector store by loading existing index or creating new one.

**Returns**: `VectorStore` (FAISS instance)

**Logic**:
```python
if os.path.exists("microwave_faiss_index"):
    vectorstore = FAISS.load_local(...)
else:
    vectorstore = self._create_new_index()
return vectorstore
```

**Side Effects**:
- Reads from `microwave_faiss_index/` directory
- Calls `_create_new_index()` if index missing

**Not intended for direct use**—called by `__init__()`.

---

#### `_create_new_index()`

Create new FAISS index from microwave manual.

**Returns**: `VectorStore` (FAISS instance)

**Process**:
1. Load `task/microwave_manual.txt`
2. Split into chunks (300 chars, 50 overlap)
3. Embed chunks using `AzureOpenAIEmbeddings`
4. Create FAISS index
5. Save to `microwave_faiss_index/`

**Parameters**: None (uses class attributes)

**Side Effects**:
- Creates `microwave_faiss_index/` directory
- Writes `index.faiss` and `index.pkl` files
- Makes multiple DIAL API calls (~30s)

**Chunking Configuration**:
```python
RecursiveCharacterTextSplitter(
    chunk_size=300,
    chunk_overlap=50,
    separators=["\n\n", "\n", "."]
)
```

**Not intended for direct use**—called by `_setup_vectorstore()` if needed.

---

## Configuration Constants

### Module: `task._constants`

```python
DIAL_URL: str = 'https://ai-proxy.lab.epam.com'
API_KEY: str = os.getenv('DIAL_API_KEY', '')
```

**Usage**:
```python
from task._constants import DIAL_URL, API_KEY

print(f"Endpoint: {DIAL_URL}")
print(f"Key configured: {bool(API_KEY)}")
```

**Security Note**: `API_KEY` sourced from environment variable. Never hardcode in version control.

---

## Prompt Templates

### System Prompt

Defined in `task/app.py`:

```python
SYSTEM_PROMPT = """You are a RAG-powered assistant that assists users with their questions about microwave usage.
            
## Structure of User message:
`RAG CONTEXT` - Retrieved documents relevant to the query.
`USER QUESTION` - The user's actual question.

## Instructions:
- Use information from `RAG CONTEXT` as context when answering the `USER QUESTION`.
- Cite specific sources when using information from the context.
- Answer ONLY based on conversation history and RAG context.
- If no relevant information exists in `RAG CONTEXT` or conversation history, state that you cannot answer the question.
"""
```

**Purpose**: Instructs LLM to:
- Stay grounded in provided context
- Avoid hallucination
- Indicate when information is unavailable

### User Prompt Template

```python
USER_PROMPT = """##RAG CONTEXT:
{context}


##USER QUESTION: 
{query}"""
```

**Variables**:
- `{context}`: Retrieved document chunks
- `{query}`: User's original question

---

## Usage Examples

### Basic Query Flow

```python
from task.app import MicrowaveRAG
from langchain_openai import AzureOpenAIEmbeddings, AzureChatOpenAI
from pydantic import SecretStr
from task._constants import DIAL_URL, API_KEY

# Initialize clients
embeddings = AzureOpenAIEmbeddings(
    deployment="text-embedding-3-small-1",
    azure_endpoint=DIAL_URL,
    api_key=SecretStr(API_KEY)
)

llm = AzureChatOpenAI(
    temperature=0.0,
    azure_deployment="gpt-4o",
    azure_endpoint=DIAL_URL,
    api_key=SecretStr(API_KEY),
    api_version=""
)

# Create RAG instance
rag = MicrowaveRAG(embeddings, llm)

# Execute query
query = "What safety precautions should I take?"
context = rag.retrieve_context(query, k=4, score=0.3)
augmented_prompt = rag.augment_prompt(query, context)
answer = rag.generate_answer(augmented_prompt)

print(f"Answer: {answer}")
```

### Custom Retrieval Parameters

```python
# Retrieve more chunks with lower threshold
context = rag.retrieve_context(
    "How does multi-stage cooking work?",
    k=8,        # Get up to 8 chunks
    score=0.2   # Lower threshold (more permissive)
)

print(f"Retrieved {len(context.split('\\n\\n'))} chunks")
```

### Batch Processing

```python
queries = [
    "How do I set the clock?",
    "What is the ECO function?",
    "How long can I cook?"
]

for query in queries:
    context = rag.retrieve_context(query)
    augmented_prompt = rag.augment_prompt(query, context)
    answer = rag.generate_answer(augmented_prompt)
    
    print(f"Q: {query}")
    print(f"A: {answer}\n")
```

### Programmatic Index Recreation

```python
import shutil

# Force rebuild of index (e.g., after updating manual)
shutil.rmtree("microwave_faiss_index", ignore_errors=True)

# Re-initialize RAG (will recreate index)
rag = MicrowaveRAG(embeddings, llm)
```

---

## Integration Patterns

### Web API Wrapper

```python
from flask import Flask, request, jsonify

app = Flask(__name__)
rag = MicrowaveRAG(embeddings, llm)

@app.route('/query', methods=['POST'])
def query():
    data = request.json
    query = data.get('query', '')
    
    context = rag.retrieve_context(query)
    augmented_prompt = rag.augment_prompt(query, context)
    answer = rag.generate_answer(augmented_prompt)
    
    return jsonify({'answer': answer})

app.run(port=5000)
```

### Async Wrapper

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=4)

async def async_query(rag, query):
    loop = asyncio.get_event_loop()
    
    context = await loop.run_in_executor(
        executor, rag.retrieve_context, query
    )
    augmented_prompt = rag.augment_prompt(query, context)
    answer = await loop.run_in_executor(
        executor, rag.generate_answer, augmented_prompt
    )
    
    return answer

# Usage
answer = await async_query(rag, "How do I clean the microwave?")
```

### Context-Only Mode (No Generation)

```python
# Use for testing retrieval without LLM costs
query = "What is the maximum cooking time?"
context = rag.retrieve_context(query, k=4, score=0.3)

print("Retrieved context:")
print(context)
# Skip augment_prompt and generate_answer
```

---

**Related Documentation**:
- [Architecture](./architecture.md) - System design and data flow
- [Testing Guide](./testing.md) - Example queries and validation
- [Setup Guide](./setup.md) - Installation and configuration
