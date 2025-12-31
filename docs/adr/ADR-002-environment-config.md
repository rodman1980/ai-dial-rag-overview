# ADR-002: Environment-Based Configuration

## Status
**Accepted** (2025-12-30)

## Context

The RAG system requires sensitive configuration values:
- DIAL API key (authentication token)
- DIAL API endpoint URL
- Azure OpenAI deployment names

### Requirements
- **Security**: API keys must not be committed to version control
- **Flexibility**: Different environments (dev, staging, prod) may use different endpoints
- **Simplicity**: Easy setup for students without complex configuration systems
- **Portability**: Configuration should work across operating systems (macOS, Linux, Windows)

### Constraints
- No external configuration management tools (Vault, Consul, etc.)
- No cloud-based secrets management
- Must work in both interactive (terminal) and programmatic contexts
- Students should understand configuration mechanism easily

## Decision

**Use environment variables with a fallback pattern in `_constants.py`.**

### Implementation

```python
# task/_constants.py
import os

DIAL_URL = 'https://ai-proxy.lab.epam.com'
API_KEY = os.getenv('DIAL_API_KEY', '')
```

### Configuration Methods

**Primary (Recommended)**: Environment variable
```bash
export DIAL_API_KEY="your-key-here"
```

**Alternative (Development only)**: Direct modification of `_constants.py` (with `.gitignore` safety net)

**Future Enhancement**: `.env` file with `python-dotenv` library (see Consequences)

## Consequences

### Positive
- **Security**: No secrets in Git history (relies on `.gitignore`)
- **Standard practice**: Follows 12-factor app methodology
- **OS-agnostic**: Environment variables work on all platforms
- **Simple setup**: One-line export command
- **No dependencies**: Uses built-in `os.getenv()`
- **Clear error messages**: Empty string default allows for explicit validation

### Negative
- **Manual setup**: Users must set environment variable before running
- **Not persistent**: Environment variables cleared on shell exit (unless in `.bashrc`/`.zshrc`)
- **No validation**: Doesn't check if API key is valid format
- **Discovery friction**: Students may not know to set environment variable

### Neutral
- **Centralized**: All configuration in one file (`_constants.py`)
- **Read-only**: Configuration not modifiable at runtime (acceptable for this use case)

## Alternatives Considered

### 1. Configuration File (config.json / config.yaml)

**Implementation**:
```python
import json
with open('config.json') as f:
    config = json.load(f)
API_KEY = config['dial_api_key']
```

**Pros**:
- More discoverable (file in repository structure)
- Can include comments and documentation
- Supports nested configuration

**Cons**:
- **Security risk**: Easy to accidentally commit secrets
- **Requires .gitignore discipline**: Must exclude from version control
- **More boilerplate**: File I/O, error handling
- **Not standard**: Environment variables more universal

**Verdict**: Rejected (higher security risk, more complex)

---

### 2. Command-Line Arguments

**Implementation**:
```bash
python -m task.app --api-key="your-key-here"
```

**Pros**:
- Explicit and visible
- No persistence issues

**Cons**:
- **Shell history exposure**: API key visible in command history
- **Verbose**: Long commands for multiple arguments
- **Integration friction**: Harder to use in scripts or notebooks

**Verdict**: Rejected (security issue with shell history)

---

### 3. Python-Dotenv (.env file)

**Implementation**:
```python
# task/_constants.py
from dotenv import load_dotenv
import os

load_dotenv()
API_KEY = os.getenv('DIAL_API_KEY', '')
```

```bash
# .env (in .gitignore)
DIAL_API_KEY=your-key-here
```

**Pros**:
- **Persistent**: No need to export on every shell session
- **Discoverable**: `.env` file pattern well-known
- **Clean separation**: Secrets in separate file
- **Development-friendly**: Easy to switch between environments

**Cons**:
- **Extra dependency**: Requires `pip install python-dotenv`
- **Not in requirements.txt**: Would need to add dependency
- **Slightly more complex**: Students need to understand `.env` pattern

**Verdict**: **Deferred** (good future enhancement, see Roadmap)

---

### 4. Secrets Management Service (AWS Secrets Manager, HashiCorp Vault)

**Pros**:
- Production-grade security
- Audit logging and rotation
- Centralized management

**Cons**:
- **Massive overkill**: For learning project with local execution
- **Infrastructure complexity**: Requires external services
- **Cost**: AWS Secrets Manager $0.40/secret/month

**Verdict**: Rejected (inappropriate for educational scope)

---

### 5. Hardcoded Values

**Implementation**:
```python
API_KEY = "sk-proj-abc123..."  # ⚠️ NEVER DO THIS
```

**Pros**:
- Zero setup friction

**Cons**:
- **Major security risk**: Secrets exposed in Git
- **No flexibility**: Can't change without code modification
- **Anti-pattern**: Violates all security best practices

**Verdict**: Rejected (unacceptable security practice)

---

## Trade-offs Analysis

| Criterion | Env Vars | Config File | CLI Args | .env File | Secrets Service |
|-----------|----------|-------------|----------|-----------|-----------------|
| **Security** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Simplicity** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ |
| **Discoverability** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Standards Compliance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Setup Friction** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ |

**Winner for this use case**: Environment variables (good balance, no extra dependencies)

---

## Implementation Notes

### Validation Strategy

Add explicit validation in application startup:

```python
from task._constants import API_KEY

if not API_KEY:
    raise ValueError(
        "DIAL_API_KEY environment variable not set. "
        "Get your key from: https://support.epam.com/..."
    )
```

### Documentation Requirements

Must document in multiple places:
- [Setup Guide](../setup.md) - Detailed setup instructions
- README.md - Quick start section
- Error messages - Helpful instructions when missing

### Platform-Specific Instructions

**Linux/macOS**:
```bash
export DIAL_API_KEY="your-key-here"
```

**Windows Command Prompt**:
```cmd
set DIAL_API_KEY=your-key-here
```

**Windows PowerShell**:
```powershell
$env:DIAL_API_KEY="your-key-here"
```

### Persistence Strategy

**Temporary (current session)**:
```bash
export DIAL_API_KEY="..."
```

**Persistent (recommended for development)**:
```bash
# Add to ~/.bashrc or ~/.zshrc
echo 'export DIAL_API_KEY="your-key-here"' >> ~/.bashrc
source ~/.bashrc
```

### .gitignore Protection

Ensure `.gitignore` includes:
```gitignore
# Environment files
.env
.env.local
.env.*.local

# Configuration files with secrets
*_constants.py.local
config.json
secrets.yaml
```

---

## Security Best Practices

1. **Never commit secrets**: Always use environment variables or external files
2. **Rotate keys regularly**: DIAL API keys should be rotated every 90 days
3. **Use least privilege**: API keys should have minimal required permissions
4. **Audit access**: Monitor API key usage for suspicious activity

### What If a Secret is Committed?

1. **Immediate rotation**: Invalidate the exposed key in DIAL portal
2. **Git history cleanup**: Use `git-filter-repo` or BFG Repo-Cleaner
3. **Notification**: Inform security team
4. **Post-mortem**: Document how it happened and prevention measures

---

## Future Enhancements

### Phase 1: Add python-dotenv Support (Low Effort)

**Timeline**: Next version  
**Effort**: 1 hour

```python
# task/_constants.py
from dotenv import load_dotenv
import os

load_dotenv()  # Load .env file if present
DIAL_URL = 'https://ai-proxy.lab.epam.com'
API_KEY = os.getenv('DIAL_API_KEY', '')
```

**Benefits**: Better developer experience, persistent configuration

---

### Phase 2: Add Configuration Validation (Medium Effort)

**Timeline**: Future version  
**Effort**: 2-3 hours

```python
def validate_config():
    if not API_KEY:
        raise ValueError("DIAL_API_KEY not set")
    if not API_KEY.startswith('sk-'):
        raise ValueError("Invalid API key format")
    if len(API_KEY) < 20:
        raise ValueError("API key too short")
```

---

### Phase 3: Multiple Environment Support (High Effort)

For production deployment scenarios:

```python
ENVIRONMENT = os.getenv('ENVIRONMENT', 'development')

CONFIG = {
    'development': {
        'dial_url': 'https://ai-proxy.lab.epam.com',
        'log_level': 'DEBUG'
    },
    'production': {
        'dial_url': 'https://ai-proxy.epam.com',
        'log_level': 'INFO'
    }
}

DIAL_URL = CONFIG[ENVIRONMENT]['dial_url']
```

---

## References

- [12-Factor App: Config](https://12factor.net/config)
- [OWASP: Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Python-Dotenv Documentation](https://pypi.org/project/python-dotenv/)

---

**Decision Made By**: Architecture team  
**Date**: 2025-12-30  
**Supersedes**: None  
**Related ADRs**: None
