# Data Model: Core Entities

**Phase**: 1 (Design)
**Date**: 2026-01-07

This document defines the core data structures for ai-comment-sweeper MVP.

---

## Configuration Models

### Config (Root Configuration)

**Purpose**: Root configuration loaded from `.aisweeper.json` with defaults.

**Fields**:
- `aggressiveness: Literal['conservative', 'moderate', 'aggressive', 'nuclear']` - Comment removal aggressiveness level (default: 'moderate')
- `llm: LLMConfig` - LLM provider configuration
- `markdown: MarkdownConfig` - Markdown processing configuration
- `cache: CacheConfig` - Cache settings
- `ignore: list[str]` - File/directory patterns to ignore (default: ['node_modules/', 'vendor/', '.git/'])

**Validation Rules**:
- Aggressiveness must be one of four valid levels
- All nested configs validated recursively
- Environment variable resolution applied

**State Transitions**: N/A (immutable after load)

---

### LLMConfig

**Purpose**: LLM provider and authentication configuration.

**Fields**:
- `provider: Literal['openai', 'anthropic', 'azure', 'ollama']` - LLM provider name (default: 'anthropic')
- `model: str` - Model identifier (default: 'claude-3-5-haiku-20241022')
- `api_key: str | None` - API key or environment variable reference (e.g., '${ANTHROPIC_API_KEY}')

**Validation Rules**:
- Provider must be supported
- If `api_key` starts with `${` and ends with `}`, resolve from environment variable
- Raise error if environment variable not set

**Relationships**:
- Parent: `Config`

---

### MarkdownConfig

**Purpose**: Configuration for markdown documentation processing.

**Fields**:
- `sacred: list[str]` - Files never to process (default: ['README.md', 'CONTRIBUTING.md', 'LICENSE.md', 'CHANGELOG.md', 'AGENTS.md'])
- `ephemeral_patterns: list[str]` - Filename patterns for ephemeral docs (default: ['*_NOTES.md', 'AI_*.md', 'SCRATCH*.md', 'TEMP*.md'])
- `ephemeral_age_days: int` - Age threshold for ephemeral detection (default: 30)
- `deletion_analysis: bool` - Whether to use LLM for deletion analysis (default: True)

**Validation Rules**:
- `ephemeral_age_days` must be positive
- Sacred files and patterns are case-sensitive

**Relationships**:
- Parent: `Config`

---

### CacheConfig

**Purpose**: Cache storage and TTL configuration.

**Fields**:
- `path: str` - Cache directory path (default: '.aisweeper-cache/')
- `ttl: int` - Time-to-live in seconds (default: 3600)

**Validation Rules**:
- `ttl` must be positive
- `path` is relative to repository root

**Relationships**:
- Parent: `Config`

---

## File Models

### SourceFile

**Purpose**: Represents a source code file to be processed for comment cleaning.

**Fields**:
- `path: Path` - Absolute file path
- `relative_path: str` - Path relative to repository root
- `language: str` - Programming language (e.g., 'python', 'typescript')
- `content: str` - Original file content
- `last_commit_time: int` - Unix timestamp of last git commit
- `content_hash: str` - SHA256 hash of content

**Relationships**:
- Processed by: `SourceProcessor`
- Results cached in: `CacheEntry`

**State Transitions**: N/A (data object)

---

### DocumentationFile

**Purpose**: Represents a markdown documentation file for deletion analysis.

**Fields**:
- `path: Path` - Absolute file path
- `relative_path: str` - Path relative to repository root
- `content: str` - File content
- `last_commit_time: int` - Unix timestamp of last git commit
- `days_since_commit: int` - Days since last commit
- `matches_ephemeral_pattern: bool` - Whether filename matches ephemeral patterns
- `is_sacred: bool` - Whether file is in sacred list
- `content_hash: str` - SHA256 hash of content

**Validation Rules**:
- `is_sacred` files must never be processed
- `days_since_commit` derived from `last_commit_time`

**Relationships**:
- Processed by: `DocumentationProcessor`
- Results cached in: `CacheEntry`

**State Transitions**: N/A (data object)

---

## Processing Results

### SourceProcessingResult

**Purpose**: Result of LLM processing for source file comment cleaning.

**Fields**:
- `action: Literal['clean', 'skip']` - Action taken ('clean' = modified, 'skip' = no changes)
- `original_content: str` - Original file content
- `cleaned_content: str | None` - Cleaned content (None if action='skip')
- `diff: str | None` - Unified diff (None if action='skip')
- `comments_removed: int` - Number of comment lines removed
- `comments_kept: int` - Number of comment lines preserved
- `tokens_used: int` - LLM tokens consumed
- `cost_estimate: float` - Estimated cost in USD

**Validation Rules**:
- If `action='clean'`, `cleaned_content` and `diff` must be non-None
- If `action='skip'`, `cleaned_content` and `diff` must be None
- `comments_removed` and `comments_kept` must be non-negative

**State Transitions**:
```
[Processing] → action='clean' → [Cleaned]
             → action='skip' → [Skipped]
```

---

### DocumentationProcessingResult

**Purpose**: Result of LLM analysis for markdown documentation deletion.

**Fields**:
- `recommendation: Literal['DELETE', 'KEEP', 'PRUNE']` - Deletion recommendation
- `reasoning: str` - Explanation for recommendation
- `confidence: Literal['high', 'medium', 'low']` - Confidence level
- `tokens_used: int` - LLM tokens consumed
- `cost_estimate: float` - Estimated cost in USD

**Validation Rules**:
- `reasoning` must be non-empty
- Only HIGH confidence DELETE recommendations are acted upon

**State Transitions**:
```
[Analysis] → recommendation='DELETE' + confidence='high' → [Will Delete]
           → recommendation='DELETE' + confidence!='high' → [Will Keep]
           → recommendation='KEEP' → [Will Keep]
           → recommendation='PRUNE' → [Will Keep] (user manual action)
```

---

## Cache Models

### CacheManifest

**Purpose**: Top-level cache metadata for invalidation.

**Fields**:
- `git_commit: str` - HEAD commit hash when cache was created
- `timestamp: int` - Unix timestamp when cache was created
- `ttl: int` - Cache time-to-live in seconds

**Validation Rules**:
- Cache is invalid if current commit != `git_commit`
- Cache is stale if `(now - timestamp) > ttl`

**Relationships**:
- Stored at: `.aisweeper-cache/manifest.json`
- References: All `CacheEntry` files

---

### CacheEntry

**Purpose**: Individual file processing result cached on disk.

**Fields**:
- `file_path: str` - Relative file path
- `original_hash: str` - SHA256 hash of original content
- `timestamp: int` - Unix timestamp when processed
- `git_commit: str` - Commit hash when processed
- `category: Literal['source', 'docs']` - File category
- `result: SourceProcessingResult | DocumentationProcessingResult` - Processing result (serialized)

**Validation Rules**:
- Entry is invalid if file content hash != `original_hash`
- Entry is invalid if current commit != `git_commit`
- Entry must match category type (source vs docs)

**Relationships**:
- Parent: `CacheManifest`
- Stored at: `.aisweeper-cache/{category}/{hash(file_path)}.json`

**State Transitions**:
```
[File Modified] → invalidate → [Entry Deleted]
[Git Commit Changed] → invalidate → [Entry Deleted]
[TTL Expired] → stale → [Entry Ignored]
```

---

## Scan Results

### ScanResult

**Purpose**: Result of repository scanning phase.

**Fields**:
- `source_files: list[SourceFile]` - Source files to process
- `documentation_files: list[DocumentationFile]` - Markdown files to analyze
- `sacred_files: list[str]` - Sacred files skipped
- `ephemeral_files: list[DocumentationFile]` - Files matching ephemeral pattern + age
- `ignored_files: list[str]` - Files ignored per config/gitignore

**Validation Rules**:
- No overlap between categories
- Sacred files never appear in `documentation_files`

**Relationships**:
- Generated by: `Scanner`
- Used by: `SourceProcessor`, `DocumentationProcessor`

---

## Summary Statistics

### ProcessingSummary

**Purpose**: Aggregate statistics for summary report.

**Fields**:
- `files_processed: int` - Total files processed
- `source_files_cleaned: int` - Source files modified
- `source_files_skipped: int` - Source files unchanged
- `docs_recommended_delete: int` - Docs recommended for deletion
- `docs_kept: int` - Docs recommended to keep
- `comments_removed: int` - Total comment lines removed
- `comments_kept: int` - Total comment lines preserved
- `total_tokens_used: int` - Total LLM tokens consumed
- `total_cost_estimate: float` - Total cost in USD
- `processing_time: float` - Total processing time in seconds

**Validation Rules**:
- All counts must be non-negative
- `files_processed = source_files_cleaned + source_files_skipped + docs_recommended_delete + docs_kept`

**Relationships**:
- Generated by: `Reporter`
- Displayed after: Processing complete

---

## Entity Relationships Diagram

```
Config
├── LLMConfig
├── MarkdownConfig
└── CacheConfig

ScanResult
├── SourceFile[] ──► SourceProcessor ──► SourceProcessingResult ──► CacheEntry
└── DocumentationFile[] ──► DocumentationProcessor ──► DocumentationProcessingResult ──► CacheEntry

CacheManifest
└── CacheEntry[]
    ├── SourceProcessingResult (for source files)
    └── DocumentationProcessingResult (for docs)

ProcessingSummary
└── Aggregates results from all CacheEntry[]
```

---

## JSON Schema Examples

### Config (.aisweeper.json)
```json
{
  "aggressiveness": "moderate",
  "llm": {
    "provider": "anthropic",
    "model": "claude-3-5-haiku-20241022",
    "api_key": "${ANTHROPIC_API_KEY}"
  },
  "markdown": {
    "sacred": ["README.md", "LICENSE.md"],
    "ephemeral_patterns": ["*_NOTES.md", "AI_*.md"],
    "ephemeral_age_days": 30,
    "deletion_analysis": true
  },
  "cache": {
    "path": ".aisweeper-cache/",
    "ttl": 3600
  },
  "ignore": ["node_modules/", "vendor/"]
}
```

### CacheEntry (source file)
```json
{
  "file_path": "src/utils/helpers.py",
  "original_hash": "a3c2f1...",
  "timestamp": 1704459600,
  "git_commit": "def456...",
  "category": "source",
  "result": {
    "action": "clean",
    "original_content": "# I'll add a helper function\ndef foo():\n    pass",
    "cleaned_content": "def foo():\n    pass",
    "diff": "--- a/src/utils/helpers.py\n+++ b/src/utils/helpers.py\n@@ -1,3 +1,2 @@\n-# I'll add a helper function\n def foo():\n     pass",
    "comments_removed": 1,
    "comments_kept": 0,
    "tokens_used": 120,
    "cost_estimate": 0.0001
  }
}
```

### CacheEntry (documentation file)
```json
{
  "file_path": "AI_NOTES.md",
  "original_hash": "b4d3e2...",
  "timestamp": 1704459600,
  "git_commit": "def456...",
  "category": "docs",
  "result": {
    "recommendation": "DELETE",
    "reasoning": "This file contains AI conversation logs from 120 days ago with no recent updates. Content appears to be scratch notes from an implementation session that is now complete.",
    "confidence": "high",
    "tokens_used": 300,
    "cost_estimate": 0.0002
  }
}
```
