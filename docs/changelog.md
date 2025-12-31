---
title: Changelog
description: Notable changes, releases, and version history
version: 1.0.0
last_updated: 2025-12-30
related: [README.md, roadmap.md]
tags: [changelog, releases, history]
---

# Changelog

All notable changes to the AI DIAL RAG Overview project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned
- Error handling and retry logic for DIAL API calls
- .env file support with python-dotenv
- Automated test suite (pytest)
- Logging and instrumentation
- Interactive tutorial mode

See [Roadmap](./roadmap.md) for full backlog.

---

## [1.0.0] - 2025-12-30

### Added

#### Core Features
- **RAG Pipeline**: Complete three-stage implementation (Retrieval, Augmentation, Generation)
- **Vector Storage**: FAISS-based similarity search with local persistence
- **Knowledge Base**: DW 395 HCG microwave manual (~497 lines)
- **CLI Interface**: Interactive command-line question-answering
- **Azure OpenAI Integration**: Via EPAM DIAL API proxy

#### Components
- `MicrowaveRAG` class with three public methods:
  - `retrieve_context()`: Similarity search with configurable k and score
  - `augment_prompt()`: Context injection into user query
  - `generate_answer()`: LLM-based response generation
- Configuration module (`_constants.py`) with environment variable support
- Vector index persistence to avoid re-embedding

#### Documentation
- Comprehensive documentation suite (7 documents)
  - [README.md](./README.md): Documentation hub and quick start
  - [Architecture](./architecture.md): System design with Mermaid diagrams
  - [Setup Guide](./setup.md): Installation and configuration
  - [API Reference](./api.md): Complete API documentation with examples
  - [Testing Guide](./testing.md): Test queries and validation strategies
  - [Glossary](./glossary.md): Domain terms and technical concepts
  - [Roadmap](./roadmap.md): Future enhancements and backlog
- Architecture Decision Records (ADRs):
  - [ADR-001](./adr/ADR-001-faiss-vector-storage.md): FAISS for vector storage
  - [ADR-002](./adr/ADR-002-environment-config.md): Environment-based configuration
  - ADR-003: Document chunking strategy (planned)
  - ADR-004: Synchronous pipeline design (planned)

#### Educational Features
- Verbose console output showing each RAG stage
- Sample test queries (10 valid, 2 invalid)
- Retrieval score visualization
- Step-by-step pipeline demonstration

### Technical Details

#### Dependencies
- `langchain-community==0.4.1`: Document loaders, vector stores
- `langchain-openai==1.0.2`: Azure OpenAI integrations
- `langchain-text-splitters==1.0.0`: Document chunking
- `faiss-cpu==1.12.0`: Vector similarity search

#### Configuration
- Embedding model: `text-embedding-3-small-1` (1536 dimensions)
- LLM model: `gpt-4o` (temperature=0.0)
- Chunking: 300 chars with 50 overlap
- Retrieval: k=4 chunks, score threshold=0.3

#### Performance
- Index creation: ~30 seconds (one-time)
- Index loading: ~2 seconds (subsequent runs)
- Query latency: ~2.5-5.5 seconds (end-to-end)
  - Retrieval: ~210ms
  - Augmentation: <1ms
  - Generation: 2-5s

### Security
- API key via environment variable (no hardcoded secrets)
- `.gitignore` patterns for sensitive files
- Documentation of security best practices

---

## [0.9.0] - 2025-12-15 (Pre-release)

### Added
- Initial project structure
- Basic document loading
- FAISS index creation
- Placeholder RAG methods (TODOs for students)

### Documentation
- README.md with project overview
- Setup instructions
- Sample test queries

### Notes
This was a skeleton project for students to complete as a learning exercise.

---

## Version History

| Version | Date | Status | Highlights |
|---------|------|--------|------------|
| 1.0.0 | 2025-12-30 | Stable | Complete RAG implementation, full documentation |
| 0.9.0 | 2025-12-15 | Pre-release | Student starter template |

---

## Upgrade Guide

### From 0.9.0 to 1.0.0

**Breaking Changes**: None (1.0.0 is complete implementation of 0.9.0 TODOs)

**New Features**:
- Fully implemented `retrieve_context()`, `augment_prompt()`, `generate_answer()`
- FAISS index persistence
- Comprehensive documentation

**Migration Steps**:
1. Pull latest code: `git pull origin main`
2. Reinstall dependencies: `pip install -r requirements.txt`
3. Set API key: `export DIAL_API_KEY="your-key"`
4. Delete old index: `rm -rf microwave_faiss_index/` (will regenerate)
5. Run: `python -m task.app`

**Deprecated**: None

---

## Contributing

When adding changes:
1. Follow [Keep a Changelog](https://keepachangelog.com/) format
2. Categorize under: Added, Changed, Deprecated, Removed, Fixed, Security
3. Include issue/PR references: `- Fixed retrieval bug (#42)`
4. Update version number in documentation front matter
5. Update "Unreleased" section with each PR
6. Create version release when merging to main

---

## Release Process

1. Update `[Unreleased]` section with changes
2. Set release date and version number
3. Update documentation `last_updated` dates
4. Create Git tag: `git tag -a v1.0.0 -m "Release 1.0.0"`
5. Push tag: `git push origin v1.0.0`
6. Create GitHub/GitLab release with changelog excerpt

---

## Semantic Versioning

**MAJOR.MINOR.PATCH** (e.g., 1.0.0)

- **MAJOR**: Incompatible API changes
- **MINOR**: New features, backwards-compatible
- **PATCH**: Bug fixes, backwards-compatible

**Pre-release labels**: `alpha`, `beta`, `rc` (e.g., 1.1.0-beta.1)

---

## Links

- [Roadmap](./roadmap.md) - Planned features
- [ADR Index](./adr/README.md) - Architecture decisions
- [Contributing Guidelines](../CONTRIBUTING.md) - How to contribute (TODO)

---

**Maintained by**: EPAM AI/ML Team  
**Format**: [Keep a Changelog](https://keepachangelog.com/)  
**Versioning**: [Semantic Versioning](https://semver.org/)
