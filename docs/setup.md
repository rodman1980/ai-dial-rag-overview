---
title: Setup Guide
description: Environment setup, dependency installation, and configuration instructions
version: 1.0.0
last_updated: 2025-12-30
related: [README.md, architecture.md, testing.md]
tags: [setup, installation, configuration, environment]
---

# Setup Guide

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### System Requirements

| Component | Requirement |
|-----------|-------------|
| **Python** | 3.11 or higher |
| **pip** | 20.0+ (bundled with Python) |
| **RAM** | 2GB minimum (for FAISS index) |
| **Disk** | 500MB free space |
| **Network** | EPAM VPN connection |
| **OS** | macOS, Linux, Windows (WSL recommended) |

### Network Access

**Required**: EPAM VPN connection to access DIAL API  
**Endpoint**: `https://ai-proxy.lab.epam.com`

**To verify VPN connectivity**:
```bash
curl -I https://ai-proxy.lab.epam.com/openai/models
# Expected: HTTP/2 401 (unauthorized, but reachable)
```

### API Key

**Get your DIAL API key**:
1. Connect to EPAM VPN
2. Visit: [https://support.epam.com/ess?id=sc_cat_item&table=sc_cat_item&sys_id=910603f1c3789e907509583bb001310c](https://support.epam.com/ess?id=sc_cat_item&table=sc_cat_item&sys_id=910603f1c3789e907509583bb001310c)
3. Follow self-service portal instructions
4. Copy the generated API key (keep secure)

## Installation

### 1. Clone Repository

```bash
# Using HTTPS
git clone https://git.epam.com/ai-dial-rag-overview.git
cd ai-dial-rag-overview

# Or using SSH
git clone git@git.epam.com:ai-dial-rag-overview.git
cd ai-dial-rag-overview
```

### 2. Create Virtual Environment (Recommended)

**Using venv (built-in)**:
```bash
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

**Using conda**:
```bash
conda create -n rag-env python=3.11
conda activate rag-env
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

**Expected output**:
```
Collecting langchain-community==0.4.1
Collecting langchain-openai==1.0.2
Collecting langchain-text-splitters==1.0.0
Collecting faiss-cpu==1.12.0
...
Successfully installed langchain-community-0.4.1 ...
```

**Verify installation**:
```bash
pip list | grep langchain
# Should show:
# langchain-community    0.4.1
# langchain-openai       1.0.2
# langchain-text-splitters 1.0.0
```

### Dependency Details

| Package | Version | Purpose |
|---------|---------|---------|
| `langchain-community` | 0.4.1 | Document loaders, vector stores |
| `langchain-openai` | 1.0.2 | Azure OpenAI integrations |
| `langchain-text-splitters` | 1.0.0 | Document chunking |
| `faiss-cpu` | 1.12.0 | Vector similarity search |

**Total install size**: ~250MB (includes NumPy, pandas, etc.)

## Configuration

### 1. Set API Key

**Option A: Environment Variable (Recommended)**

**Linux/macOS**:
```bash
export DIAL_API_KEY="your-api-key-here"
```

**Windows (Command Prompt)**:
```cmd
set DIAL_API_KEY=your-api-key-here
```

**Windows (PowerShell)**:
```powershell
$env:DIAL_API_KEY="your-api-key-here"
```

**Option B: Modify `_constants.py` (Development Only)**

⚠️ **Warning**: Never commit API keys to version control

```python
# task/_constants.py
import os

DIAL_URL = 'https://ai-proxy.lab.epam.com'
API_KEY = 'your-api-key-here'  # Only for local testing
```

**Best Practice**: Use `.env` file with `python-dotenv`:
```bash
# .env (add to .gitignore)
DIAL_API_KEY=your-api-key-here
```

```python
# task/_constants.py
import os
from dotenv import load_dotenv

load_dotenv()
DIAL_URL = 'https://ai-proxy.lab.epam.com'
API_KEY = os.getenv('DIAL_API_KEY', '')
```

### 2. Verify Configuration

**Check API key is set**:
```bash
python -c "from task._constants import API_KEY; print('✓ API key configured' if API_KEY else '✗ Missing API key')"
```

**Test DIAL API connectivity**:
```bash
python -c "
import os
import httpx
from task._constants import DIAL_URL, API_KEY

headers = {'Authorization': f'Bearer {API_KEY}'}
response = httpx.get(f'{DIAL_URL}/openai/models', headers=headers)
print(f'Status: {response.status_code}')
print('✓ DIAL API accessible' if response.status_code == 200 else '✗ Connection failed')
"
```

**Expected output**:
```
Status: 200
✓ DIAL API accessible
```

### 3. Verify Knowledge Base

```bash
ls -lh task/microwave_manual.txt
# Expected: ~25KB file
```

**View first 10 lines**:
```bash
head -n 10 task/microwave_manual.txt
```

## Verification

### Run the Application

```bash
python -m task.app
```

**Expected first-run output**:
```
🔄 Initializing Microwave Manual RAG System...
📂 No existing FAISS index found. Creating new index...
📖 Loading text document...
🎯 Microwave RAG Assistant

> 
```

**Index creation time**: ~30 seconds (one-time)

**Subsequent runs**:
```
🔄 Initializing Microwave Manual RAG System...
📂 Existing FAISS index found. Loading...
🎯 Microwave RAG Assistant

> 
```

**Loading time**: <2 seconds

### Test Query

```bash
> What safety precautions should be taken to avoid exposure to excessive microwave energy?
```

**Expected output structure**:
```
====================================================================================================
🔍 STEP 1: RETRIEVAL
----------------------------------------------------------------------------------------------------
Query: 'What safety precautions should be taken to avoid exposure to excessive microwave energy?'
Searching for top 4 most relevant chunks with similarity score 0.3:
Score: 0.85
Content: PRECAUTIONS TO AVOID POSSIBLE EXPOSURE TO EXCESSIVE MICROWAVE ENERGY...
...
====================================================================================================

🔗 STEP 2: AUGMENTATION
----------------------------------------------------------------------------------------------------
Context: ...
Question: What safety precautions...
====================================================================================================

🤖 STEP 3: GENERATION
----------------------------------------------------------------------------------------------------
Response: To avoid exposure to excessive microwave energy, you should...

Answer: To avoid exposure to excessive microwave energy, you should...
```

### Verify FAISS Index Created

```bash
ls -lh microwave_faiss_index/
# Expected output:
# -rw-r--r--  1 user  staff   123K Dec 30 10:00 index.faiss
# -rw-r--r--  1 user  staff    45K Dec 30 10:00 index.pkl
```

## Troubleshooting

### Common Issues

#### 1. `ModuleNotFoundError: No module named 'langchain_community'`

**Cause**: Dependencies not installed or wrong virtual environment

**Solution**:
```bash
# Verify you're in the correct environment
which python
# Should point to your venv/bin/python

# Reinstall dependencies
pip install -r requirements.txt
```

#### 2. `FileNotFoundError: The file 'microwave_manual.txt' does not exist`

**Cause**: Running script from wrong directory or missing knowledge base

**Solution**:
```bash
# Ensure you're in project root
pwd
# Should end with: /ai-dial-rag-overview

# Verify file exists
ls task/microwave_manual.txt

# If missing, check Git status
git status
git checkout task/microwave_manual.txt
```

#### 3. `401 Unauthorized` from DIAL API

**Cause**: Missing or invalid API key

**Solution**:
```bash
# Check API key is set
echo $DIAL_API_KEY

# If empty, set it
export DIAL_API_KEY="your-key-here"

# Verify in Python
python -c "from task._constants import API_KEY; print(API_KEY[:10] + '...' if API_KEY else 'MISSING')"
```

#### 4. `Connection Error` to DIAL API

**Cause**: Not connected to EPAM VPN

**Solution**:
1. Connect to EPAM VPN
2. Test connectivity:
   ```bash
   curl -I https://ai-proxy.lab.epam.com/openai/models
   ```
3. If still failing, check proxy settings:
   ```bash
   echo $HTTP_PROXY
   echo $HTTPS_PROXY
   ```

#### 5. `RuntimeError: FAISS index deserialization failed`

**Cause**: Corrupted index or version mismatch

**Solution**:
```bash
# Delete corrupted index
rm -rf microwave_faiss_index/

# Restart application (will recreate index)
python -m task.app
```

#### 6. Slow query responses (>10s)

**Cause**: Network latency to DIAL API

**Solution**:
- Verify VPN connection stability
- Check DIAL API status: TODO: Add status page link
- Consider caching strategies (see [Roadmap](./roadmap.md))

### Performance Tuning

**Adjust retrieval parameters**:
```python
# In task/app.py, line ~151
context = rag.retrieve_context(
    user_question,
    k=4,        # Number of chunks (2-8 recommended)
    score=0.3   # Similarity threshold (0.2-0.5 range)
)
```

**Adjust chunking parameters**:
```python
# In task/app.py, line ~68
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=300,     # 200-500 recommended
    chunk_overlap=50,   # 10-100 recommended
    separators=["\n\n", "\n", "."]
)
```

See [Architecture: Design Decisions](./architecture.md#design-decisions) for trade-offs.

### Debug Mode

**Enable verbose logging**:
```python
# Add to top of task/app.py
import logging
logging.basicConfig(level=logging.DEBUG)
```

**Inspect intermediate outputs**:
```python
# Modify main() to print intermediate results
context = rag.retrieve_context(user_question)
print(f"DEBUG: Context length = {len(context)} chars")

augmented_prompt = rag.augment_prompt(user_question, context)
print(f"DEBUG: Prompt length = {len(augmented_prompt)} chars")
```

## Next Steps

1. **Run test queries**: See [Testing Guide](./testing.md)
2. **Explore code**: Review [API Reference](./api.md)
3. **Understand design**: Read [Architecture](./architecture.md)
4. **Modify behavior**: Experiment with parameters above

---

**Need Help?**  
- Architecture questions: See [Architecture](./architecture.md)  
- API usage: See [API Reference](./api.md)  
- Domain concepts: See [Glossary](./glossary.md)
