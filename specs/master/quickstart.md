# Developer Quickstart Guide

**Last Updated**: 2026-01-07

This guide helps developers get started with ai-comment-sweeper development.

---

## Prerequisites

- Python 3.11 or higher
- Git (repository required for tool to work)
- API key for at least one LLM provider (Anthropic, OpenAI, Azure, or Ollama)

---

## Setup

### 1. Clone and Setup Environment

```bash
git clone <repository-url>
cd ai-comment-sweeper

# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Linux/Mac:
source venv/bin/activate
# On Windows:
venv\Scripts\activate

# Install dependencies
pip install -e ".[dev]"
```

### 2. Set API Key

```bash
# Anthropic (recommended for MVP)
export ANTHROPIC_API_KEY="sk-ant-..."

# Or OpenAI
export OPENAI_API_KEY="sk-..."

# Or create .env file (not committed to git)
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env
```

### 3. Install Pre-commit Hooks

```bash
pre-commit install
```

---

## Project Structure

```
ai-comment-sweeper/
├── src/ai_comment_sweeper/       # Main package
│   ├── cli.py                    # CLI entry point
│   ├── models/                   # Pydantic data models
│   ├── processors/               # Source & doc processors
│   └── llm/                      # LLM abstraction
├── tests/                        # Test suite
│   ├── unit/                     # Unit tests
│   ├── integration/              # Integration tests
│   └── fixtures/                 # Test data
├── specs/master/                 # Design documentation
├── pyproject.toml                # Project config
└── README.md                     # User documentation
```

---

## Development Workflow

### Running Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=ai_comment_sweeper --cov-report=html

# Run specific test file
pytest tests/unit/test_config.py

# Run specific test
pytest tests/unit/test_config.py::test_load_default_config
```

### Code Quality

```bash
# Format code
black src/ tests/

# Lint code
ruff check src/ tests/

# Type check
mypy src/

# Run all checks (what pre-commit does)
pre-commit run --all-files
```

### Running the Tool

```bash
# Install in editable mode (if not done already)
pip install -e .

# Run dry-run mode
ai-comment-sweeper --dry-run

# Run analysis (creates cache)
ai-comment-sweeper

# Apply cached changes
ai-comment-sweeper --apply
```

---

## Key Components

### 1. Configuration System

**File**: `src/ai_comment_sweeper/models/config.py`

```python
from ai_comment_sweeper.config import load_config

# Load config from .aisweeper.json
config = load_config()

# Access settings
print(config.aggressiveness)  # 'moderate'
print(config.llm.provider)    # 'anthropic'
```

**Default config** is used if `.aisweeper.json` doesn't exist.

### 2. LLM Client

**File**: `src/ai_comment_sweeper/llm/client.py`

```python
from ai_comment_sweeper.llm.client import LLMClient

# Create client from config
client = LLMClient(
    provider=config.llm.provider,
    model=config.llm.model,
    api_key=config.llm.api_key
)

# Make completion request
response = client.complete(
    prompt="Clean this code...",
    system_prompt="You are a code curator..."
)

# Count tokens
token_count = client.count_tokens("Some text")
```

**Providers supported**: OpenAI, Anthropic, Azure, Ollama (via LiteLLM).

### 3. Git Integration

**File**: `src/ai_comment_sweeper/safety.py`

```python
from ai_comment_sweeper.safety import get_repo, verify_git_repository

# Verify git repo (exits if not)
verify_git_repository()

# Get repo object
repo = get_repo()

# Check for uncommitted changes
if repo.is_dirty():
    print("Warning: uncommitted changes")

# Get current commit hash (for cache)
commit_hash = repo.head.commit.hexsha
```

### 4. File Scanning

**File**: `src/ai_comment_sweeper/scanner.py`

```python
from ai_comment_sweeper.scanner import scan_repository

# Scan repository
scan_result = scan_repository(config)

# Access results
print(f"Source files: {len(scan_result.source_files)}")
print(f"Docs: {len(scan_result.documentation_files)}")
print(f"Sacred files: {scan_result.sacred_files}")
```

**Scanner respects** `.gitignore` and config `ignore` patterns.

### 5. Processing Files

**File**: `src/ai_comment_sweeper/processors/source.py`

```python
from ai_comment_sweeper.processors.source import SourceProcessor

processor = SourceProcessor(config, llm_client)

# Process source file
result = processor.process(source_file)

if result.action == 'clean':
    print(f"Removed {result.comments_removed} comments")
    print(result.diff)
elif result.action == 'skip':
    print("No changes needed")
```

### 6. Caching

**File**: `src/ai_comment_sweeper/cache.py`

```python
from ai_comment_sweeper.cache import CacheManager

cache = CacheManager(
    cache_dir=Path(config.cache.path),
    git_commit=repo.head.commit.hexsha
)

# Write cache entry
cache.write_entry(entry, category='source')

# Read cache entry (validates git commit + content hash)
entry = cache.read_entry(file_path, category='source')

if entry is None:
    print("Cache miss or invalid")
```

---

## Testing Guide

### Unit Tests

**Test file naming**: `test_<module>.py`

**Example** (`tests/unit/test_config.py`):
```python
import pytest
from ai_comment_sweeper.config import load_config, Config

def test_load_default_config():
    """Test loading default configuration."""
    config = Config()
    assert config.aggressiveness == 'moderate'
    assert config.llm.provider == 'anthropic'

def test_env_var_resolution(monkeypatch):
    """Test environment variable resolution in API key."""
    monkeypatch.setenv('TEST_API_KEY', 'test-key-123')

    config_data = {
        'llm': {'api_key': '${TEST_API_KEY}'}
    }
    config = Config(**config_data)
    assert config.llm.api_key == 'test-key-123'
```

### Integration Tests

**Example** (`tests/integration/test_dry_run.py`):
```python
from click.testing import CliRunner
from ai_comment_sweeper.cli import main

def test_dry_run_mode(tmp_git_repo):
    """Test dry-run mode on sample repository."""
    runner = CliRunner()

    result = runner.invoke(main, ['--dry-run'], cwd=tmp_git_repo)

    assert result.exit_code == 0
    assert 'Estimated cost' in result.output
    assert 'No files will be modified' in result.output
```

### Fixtures

**File**: `tests/conftest.py`

```python
import pytest
from pathlib import Path
import git

@pytest.fixture
def tmp_git_repo(tmp_path):
    """Create temporary git repository for testing."""
    repo = git.Repo.init(tmp_path)

    # Add sample files
    (tmp_path / 'test.py').write_text(
        '# AI: I added this function\ndef foo():\n    pass'
    )

    repo.index.add(['test.py'])
    repo.index.commit('Initial commit')

    return tmp_path

@pytest.fixture
def mock_llm_response():
    """Mock LLM response for testing."""
    return "def foo():\n    pass"  # Cleaned version
```

---

## Common Development Tasks

### Adding a New Aggressiveness Level

1. Update `Config` model in `src/ai_comment_sweeper/models/config.py`:
   ```python
   aggressiveness: Literal['conservative', 'moderate', 'aggressive', 'nuclear', 'custom']
   ```

2. Add prompt template in `specs/master/contracts/source-processor-prompts.md`

3. Update `SourceProcessor` to handle new level

4. Add tests in `tests/unit/test_processors.py`

### Adding a New LLM Provider

1. Update `LLMConfig` model:
   ```python
   provider: Literal['openai', 'anthropic', 'azure', 'ollama', 'custom']
   ```

2. Update LiteLLM integration in `src/ai_comment_sweeper/llm/client.py`

3. Add provider-specific tests

4. Update documentation

### Adding New File Type Support

1. Update `EXTENSION_MAP` in `src/ai_comment_sweeper/scanner.py`:
   ```python
   EXTENSION_MAP = {
       '.py': 'python',
       '.new': 'newlang',  # Add here
   }
   ```

2. Add language-specific prompt adjustments in processors

3. Add tests with sample files

---

## Debugging Tips

### Enable Verbose Logging

```python
import logging

logging.basicConfig(level=logging.DEBUG)
```

### Print LLM Prompts

```python
# In src/ai_comment_sweeper/llm/client.py
def complete(self, prompt, system_prompt):
    print(f"SYSTEM: {system_prompt}")
    print(f"USER: {prompt}")
    # ... rest of code
```

### Inspect Cache

```bash
# Cache is JSON files
cat .aisweeper-cache/manifest.json
cat .aisweeper-cache/source/<hash>.json
```

### Test Single File

```python
# Create minimal test script
from pathlib import Path
from ai_comment_sweeper.config import load_config
from ai_comment_sweeper.llm.client import LLMClient
from ai_comment_sweeper.processors.source import SourceProcessor

config = load_config()
client = LLMClient(config.llm.provider, config.llm.model, config.llm.api_key)
processor = SourceProcessor(config, client)

file_content = Path('test.py').read_text()
result = processor.process_content(file_content, 'python')

print(result.diff)
```

---

## Contributing Guidelines

1. **Create feature branch**: `git checkout -b feature/my-feature`
2. **Write tests first**: TDD approach per constitution
3. **Follow code style**: Black + Ruff + mypy strict
4. **Update docs**: If adding features, update relevant .md files
5. **Run pre-commit**: `pre-commit run --all-files`
6. **Create PR**: With clear description and test evidence

---

## Getting Help

- **Documentation**: Check `specs/master/` for design docs
- **Constitution**: See `.specify/memory/constitution.md` for principles
- **Architecture**: See `ARCHITECTURE.md` for system design
- **Issues**: Create GitHub issue with reproduction steps

---

## Quick Reference

```bash
# Setup
pip install -e ".[dev]"
export ANTHROPIC_API_KEY="..."
pre-commit install

# Development
pytest                          # Run tests
black src/ tests/              # Format
ruff check src/ tests/         # Lint
mypy src/                      # Type check

# Run tool
ai-comment-sweeper --dry-run   # Estimate
ai-comment-sweeper             # Analyze
ai-comment-sweeper --apply     # Apply

# Debug
pytest -v -s                   # Verbose with stdout
pytest --pdb                   # Drop into debugger on failure
```
