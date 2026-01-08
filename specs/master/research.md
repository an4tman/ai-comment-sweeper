# Research: MVP Technology Decisions

**Phase**: 0 (Research)
**Date**: 2026-01-07
**Status**: Complete

This document consolidates research findings for technology choices in the ai-comment-sweeper MVP.

## 1. LiteLLM Integration for Multi-Provider LLM Support

### Decision
Use LiteLLM library with custom wrapper class for unified multi-provider interface.

### Rationale
- LiteLLM provides native support for OpenAI, Anthropic, Azure OpenAI, and Ollama with consistent API
- Built-in retry logic, streaming support, and cost tracking
- Handles provider-specific quirks (different response formats, rate limits)
- Active development and good documentation

### Alternatives Considered
- **Custom abstraction from scratch**: Too much maintenance burden, would need to track API changes for each provider
- **Direct provider SDKs**: Would require separate implementation for each provider, violating LLM-agnostic principle
- **LangChain**: Overly complex for our use case, adds unnecessary dependencies

### Implementation Notes
```python
from litellm import completion, token_counter
import os

class LLMClient:
    def __init__(self, provider: str, model: str, api_key: str | None = None):
        self.provider = provider
        self.model = f"{provider}/{model}"  # e.g., "anthropic/claude-3-haiku"
        if api_key:
            os.environ[f"{provider.upper()}_API_KEY"] = api_key

    def complete(self, prompt: str, system_prompt: str) -> str:
        response = completion(
            model=self.model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": prompt}
            ],
            max_retries=3,
            timeout=60.0
        )
        return response.choices[0].message.content

    def count_tokens(self, text: str) -> int:
        return token_counter(model=self.model, text=text)
```

**Key Patterns**:
- Use `completion()` not model-specific functions
- Set API keys via environment variables
- Leverage built-in retry logic
- Token counting via `token_counter()`

---

## 2. GitPython for Repository Operations

### Decision
Use GitPython for all git operations (commit hash, file history, uncommitted changes detection).

### Rationale
- Pythonic interface to git, no shell command parsing needed
- Cross-platform (works on Windows, Linux, macOS)
- Good error handling and exceptions
- Well-maintained, used by many tools

### Alternatives Considered
- **Subprocess + git CLI**: Fragile, platform-dependent, harder to test
- **Dulwich**: Pure Python git implementation, but less feature-complete and slower
- **PyGit2** (libgit2 bindings): Fast but requires C dependencies, harder to install

### Implementation Notes
```python
from git import Repo, GitCommandError
from pathlib import Path
import os

def get_repo(path: Path = Path.cwd()) -> Repo:
    """Get git repo, exit if not a git repository."""
    try:
        return Repo(path, search_parent_directories=True)
    except Exception:
        print("ERROR: Not a git repository")
        sys.exit(1)

def get_file_last_commit_time(repo: Repo, file_path: Path) -> int:
    """Get file's last commit timestamp (Unix time)."""
    try:
        commits = list(repo.iter_commits(paths=str(file_path), max_count=1))
        if commits:
            return commits[0].committed_date
        # Fallback to file mtime if no git history
        return int(file_path.stat().st_mtime)
    except GitCommandError:
        return int(file_path.stat().st_mtime)

def get_current_commit_hash(repo: Repo) -> str:
    """Get HEAD commit hash for cache invalidation."""
    return repo.head.commit.hexsha

def has_uncommitted_changes(repo: Repo) -> bool:
    """Check for uncommitted changes (warning, not blocking)."""
    return repo.is_dirty(untracked_files=True)
```

**Key Patterns**:
- Use `search_parent_directories=True` to find repo root
- Always have fallbacks for edge cases (shallow clones, new files)
- Use `repo.is_dirty()` for uncommitted change detection
- Cache Repo object, don't recreate for each operation

---

## 3. Click for CLI Framework

### Decision
Use Click for CLI argument parsing and command structure.

### Rationale
- De facto standard for Python CLI tools
- Clean decorator-based API
- Built-in help generation, validation, and error handling
- Supports flags, options, arguments, and command groups
- Good testing utilities

### Alternatives Considered
- **argparse** (stdlib): More verbose, less Pythonic
- **Typer**: Built on Click, adds type hints but newer/less mature
- **docopt**: Interesting but less flexible for complex CLIs

### Implementation Notes
```python
import click
from pathlib import Path

@click.command()
@click.option('--dry-run', is_flag=True, help='Estimate cost without processing')
@click.option('--apply', is_flag=True, help='Apply cached changes to files')
@click.option('--markdown-only', is_flag=True, help='Process only markdown files')
@click.option('--config', type=click.Path(exists=True), default='.aisweeper.json')
@click.option('--aggressiveness', type=click.Choice(['conservative', 'moderate', 'aggressive', 'nuclear']))
@click.version_option(version='0.1.0')
def main(dry_run: bool, apply: bool, markdown_only: bool, config: str, aggressiveness: str | None):
    """AI Comment Sweeper - Remove AI-generated meta-commentary from code."""

    # Validate mutually exclusive flags
    if dry_run and apply:
        raise click.UsageError("Cannot use --dry-run and --apply together")

    if apply and not cache_exists():
        raise click.ClickException("Cache not found. Run analysis first.")

    # Rest of logic...

if __name__ == '__main__':
    main()
```

**Key Patterns**:
- Use `is_flag=True` for boolean flags
- Use `click.Choice()` for enums
- Use `click.Path(exists=True)` for file validation
- Raise `click.UsageError` for argument validation
- Raise `click.ClickException` for runtime errors (shows nice formatted error)

---

## 4. Rich for Terminal Output

### Decision
Use Rich library for colorized diffs, progress bars, and formatted output.

### Rationale
- Best-in-class terminal formatting for Python
- Built-in diff syntax highlighting
- Progress bars, tables, panels, and more
- Cross-platform color support
- Markdown rendering capabilities

### Alternatives Considered
- **colorama**: Basic color support, no advanced formatting
- **termcolor**: Similar to colorama, limited features
- **Manual ANSI codes**: Not cross-platform, hard to maintain

### Implementation Notes
```python
from rich.console import Console
from rich.table import Table
from rich.syntax import Syntax
from rich.progress import track
from rich.panel import Panel
from rich import print as rprint

console = Console()

def display_diff(diff_text: str, file_path: str):
    """Display colorized diff."""
    console.print(f"\n[bold blue]{file_path}[/bold blue]")
    syntax = Syntax(diff_text, "diff", theme="monokai", line_numbers=False)
    console.print(syntax)

def display_summary(stats: dict):
    """Display summary table."""
    table = Table(title="Summary Report")
    table.add_column("Metric", style="cyan")
    table.add_column("Value", style="magenta")

    table.add_row("Files Processed", str(stats['files_processed']))
    table.add_row("Comments Removed", str(stats['comments_removed']))
    table.add_row("Comments Kept", str(stats['comments_kept']))
    table.add_row("Docs Deleted", str(stats['docs_deleted']))

    console.print(table)

def show_safety_warning():
    """Display safety banner."""
    warning = Panel(
        "[bold red]⚠️  WARNING[/bold red]: This tool modifies and deletes files!\n"
        "Only use in a version-controlled repository.\n"
        "Commit your work before running with --apply.",
        border_style="red"
    )
    console.print(warning)

# Progress bars
for file in track(files, description="Processing files..."):
    process_file(file)
```

**Key Patterns**:
- Use `Syntax` for code/diff highlighting
- Use `Table` for structured data
- Use `Panel` for warnings/alerts
- Use `track()` for progress bars
- Console object is reusable

---

## 5. Pydantic for Configuration Validation

### Decision
Use Pydantic v2 for `.aisweeper.json` schema validation and environment variable resolution.

### Rationale
- Industry standard for data validation in Python
- Type-safe configuration with excellent error messages
- Built-in environment variable resolution
- Nested model support
- JSON schema generation

### Alternatives Considered
- **dataclasses + manual validation**: Too much boilerplate
- **attrs**: Good but less feature-rich than Pydantic
- **marshmallow**: More for serialization, less for validation

### Implementation Notes
```python
from pydantic import BaseModel, Field, field_validator
from pydantic_settings import BaseSettings
from typing import Literal
import os
import re

class LLMConfig(BaseModel):
    provider: Literal['openai', 'anthropic', 'azure', 'ollama'] = 'anthropic'
    model: str = 'claude-3-5-haiku-20241022'
    api_key: str | None = None

class MarkdownConfig(BaseModel):
    sacred: list[str] = Field(default_factory=lambda: [
        'README.md', 'CONTRIBUTING.md', 'LICENSE.md', 'CHANGELOG.md', 'AGENTS.md'
    ])
    ephemeral_patterns: list[str] = Field(default_factory=lambda: [
        '*_NOTES.md', 'AI_*.md', 'SCRATCH*.md', 'TEMP*.md'
    ])
    ephemeral_age_days: int = 30
    deletion_analysis: bool = True

class CacheConfig(BaseModel):
    path: str = '.aisweeper-cache/'
    ttl: int = 3600  # 1 hour

class Config(BaseSettings):
    aggressiveness: Literal['conservative', 'moderate', 'aggressive', 'nuclear'] = 'moderate'
    llm: LLMConfig = Field(default_factory=LLMConfig)
    markdown: MarkdownConfig = Field(default_factory=MarkdownConfig)
    cache: CacheConfig = Field(default_factory=CacheConfig)
    ignore: list[str] = Field(default_factory=lambda: ['node_modules/', 'vendor/', '.git/'])

    @field_validator('llm')
    def resolve_env_vars(cls, v):
        """Resolve ${VAR} patterns in API key."""
        if v.api_key and v.api_key.startswith('${') and v.api_key.endswith('}'):
            var_name = v.api_key[2:-1]
            v.api_key = os.getenv(var_name)
            if not v.api_key:
                raise ValueError(f"Environment variable {var_name} not set")
        return v

def load_config(path: Path = Path('.aisweeper.json')) -> Config:
    """Load and validate configuration."""
    if path.exists():
        with open(path) as f:
            data = json.load(f)
        return Config(**data)
    return Config()  # Use defaults
```

**Key Patterns**:
- Use `Literal` for enum-like fields
- Use `Field(default_factory=...)` for mutable defaults
- Use `@field_validator` for custom validation
- Use nested models for organization
- `Config(**data)` handles validation and type coercion

---

## 6. Diff Generation with difflib

### Decision
Use Python stdlib `difflib` for generating unified diffs, colorize with Rich.

### Rationale
- Built-in, no external dependency
- Generates standard unified diff format
- Works with strings or files
- Cross-platform

### Alternatives Considered
- **Subprocess + git diff**: Requires git, not portable
- **External diff tools**: Platform-dependent
- **diff-match-patch**: Overkill for our needs

### Implementation Notes
```python
import difflib
from pathlib import Path
from rich.syntax import Syntax
from rich.console import Console

def generate_diff(original: str, modified: str, file_path: str) -> str:
    """Generate unified diff."""
    original_lines = original.splitlines(keepends=True)
    modified_lines = modified.splitlines(keepends=True)

    diff = difflib.unified_diff(
        original_lines,
        modified_lines,
        fromfile=f"a/{file_path}",
        tofile=f"b/{file_path}",
        lineterm=''
    )

    return ''.join(diff)

def display_diff(diff_text: str, file_path: str):
    """Display colorized diff."""
    console = Console()
    console.print(f"\n[bold]{file_path}[/bold]")
    if diff_text:
        syntax = Syntax(diff_text, "diff", theme="monokai")
        console.print(syntax)
    else:
        console.print("[dim]No changes[/dim]")

def parse_diff_stats(diff_text: str) -> tuple[int, int]:
    """Parse diff to count added/removed lines."""
    added = sum(1 for line in diff_text.split('\n') if line.startswith('+') and not line.startswith('+++'))
    removed = sum(1 for line in diff_text.split('\n') if line.startswith('-') and not line.startswith('---'))
    return added, removed
```

**Key Patterns**:
- Use `splitlines(keepends=True)` to preserve line endings
- Use `unified_diff()` for git-style diffs
- Colorize with Rich `Syntax` class
- Parse stats by counting +/- prefixes

---

## 7. JSON-Based Cache Strategy

### Decision
Use JSON files for cache storage with SHA256 content hashing and atomic writes.

### Rationale
- Human-readable for debugging
- No database dependency
- Atomic writes via temp file + rename
- Git-based invalidation ensures correctness

### Alternatives Considered
- **SQLite**: Overkill for simple key-value storage, adds dependency
- **Pickle**: Not human-readable, security concerns
- **YAML**: Slower to parse, less standard

### Implementation Notes
```python
import json
import hashlib
from pathlib import Path
from datetime import datetime
import tempfile
import os

def hash_content(content: str) -> str:
    """SHA256 hash of file content."""
    return hashlib.sha256(content.encode()).hexdigest()

def hash_file_path(path: str) -> str:
    """Hash file path for cache filename."""
    return hashlib.sha256(path.encode()).hexdigest()

class CacheEntry(BaseModel):
    file_path: str
    original_hash: str
    timestamp: int
    git_commit: str
    result: dict  # ProcessingResult serialized

class CacheManager:
    def __init__(self, cache_dir: Path, git_commit: str):
        self.cache_dir = cache_dir
        self.git_commit = git_commit
        self.cache_dir.mkdir(exist_ok=True)
        (self.cache_dir / 'source').mkdir(exist_ok=True)
        (self.cache_dir / 'docs').mkdir(exist_ok=True)

    def write_entry(self, entry: CacheEntry, category: str):
        """Atomically write cache entry."""
        cache_file = self.cache_dir / category / f"{hash_file_path(entry.file_path)}.json"

        # Write to temp file first
        with tempfile.NamedTemporaryFile(mode='w', delete=False, dir=cache_file.parent) as tf:
            json.dump(entry.dict(), tf, indent=2)
            temp_path = tf.name

        # Atomic rename
        os.replace(temp_path, cache_file)

    def read_entry(self, file_path: str, category: str) -> CacheEntry | None:
        """Read cache entry if valid."""
        cache_file = self.cache_dir / category / f"{hash_file_path(file_path)}.json"

        if not cache_file.exists():
            return None

        with open(cache_file) as f:
            data = json.load(f)

        entry = CacheEntry(**data)

        # Validate git commit matches
        if entry.git_commit != self.git_commit:
            return None

        return entry
```

**Key Patterns**:
- Use temp file + `os.replace()` for atomic writes
- Hash file paths for cache filenames
- Validate git commit on read
- Separate source/ and docs/ subdirectories

---

## 8. File Type Detection

### Decision
Use file extension mapping with fallback to `pygments` lexer detection.

### Rationale
- Extension-based detection is fast and sufficient for 99% of cases
- Pygments provides comprehensive language detection for edge cases
- Already available (Rich depends on Pygments)

### Alternatives Considered
- **python-magic** (libmagic): Overkill, needs C dependencies
- **Extension-only**: Fails on files without extensions
- **Content-based parsing**: Too slow, complex

### Implementation Notes
```python
from pathlib import Path
from pygments.lexers import get_lexer_for_filename, ClassNotFound

# Extension to language mapping
EXTENSION_MAP = {
    '.py': 'python',
    '.js': 'javascript',
    '.ts': 'typescript',
    '.tsx': 'typescript',
    '.jsx': 'javascript',
    '.go': 'go',
    '.rs': 'rust',
    '.java': 'java',
    '.c': 'c',
    '.cpp': 'cpp',
    '.h': 'c',
    '.hpp': 'cpp',
    '.md': 'markdown',
}

def get_file_language(file_path: Path) -> str | None:
    """Detect file language by extension, fallback to pygments."""
    ext = file_path.suffix.lower()

    # Try extension mapping first
    if ext in EXTENSION_MAP:
        return EXTENSION_MAP[ext]

    # Fallback to pygments
    try:
        lexer = get_lexer_for_filename(file_path.name)
        return lexer.name.lower()
    except ClassNotFound:
        return None

def is_source_file(file_path: Path) -> bool:
    """Check if file is a source code file."""
    lang = get_file_language(file_path)
    return lang is not None and lang != 'markdown'

def is_documentation_file(file_path: Path) -> bool:
    """Check if file is markdown documentation."""
    return file_path.suffix.lower() == '.md'
```

**Key Patterns**:
- Fast path: extension mapping
- Slow path: pygments detection
- Clear categorization: source vs documentation
- None return value for unknown types

---

## Dependency Summary

**Production Dependencies** (from pyproject.toml):
```toml
dependencies = [
    "click>=8.1.0",          # CLI framework
    "litellm>=1.0.0",        # Multi-provider LLM
    "gitpython>=3.1.0",      # Git operations
    "rich>=13.0.0",          # Terminal output
    "pydantic>=2.0.0",       # Config validation
    "pydantic-settings>=2.0.0",  # Settings management
    "pygments>=2.16.0",      # Syntax detection (via Rich)
]
```

**Development Dependencies**:
```toml
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "pytest-mock>=3.12.0",   # For mocking LLM calls
    "black>=23.0.0",
    "ruff>=0.1.0",
    "mypy>=1.7.0",
    "pre-commit>=3.5.0",
]
```

---

## Open Questions for Phase 1

1. **Prompt Engineering**: What are the optimal prompts for each aggressiveness level? (Addressed in contracts/)
2. **Token Limits**: How to handle files that exceed LLM context windows? (Chunking strategy?)
3. **Parallelization**: Should we process files in parallel? (Future optimization)
4. **Progress Estimation**: How to accurately estimate processing time? (Token count × avg speed?)
5. **Error Recovery**: What happens if LLM returns invalid JSON for doc analysis? (Retry? Skip?)

---

## Implementation Priority

Based on ROADMAP.md phases, implement in this order:

**Phase 1 (Week 1-2)**: Foundation
1. Click CLI framework
2. GitPython integration
3. Pydantic config loading
4. File type detection

**Phase 2 (Week 3-4)**: Core Processing
1. LiteLLM client wrapper
2. difflib diff generation
3. JSON cache manager
4. Rich terminal output

**Phase 3 (Week 5-6)**: Polish
1. Error handling refinement
2. Progress bars
3. Summary reporting
4. Testing with real codebases
