# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`ai-comment-sweeper` is a CLI tool that removes AI-generated meta-commentary, stale comments, and ephemeral documentation from codebases developed with AI assistance. The tool uses LLM calls to intelligently clean source code comments and analyze documentation files for deletion.

## Core Architecture Principles

### 1. Two-Track Processing Model

The system processes files differently based on type:

- **Source files** (`.js`, `.ts`, `.py`, etc.): LLM cleans AI meta-commentary while preserving meaningful comments
- **Documentation files** (`.md`): LLM analyzes for deletion recommendation with confidence levels (DELETE/KEEP/PRUNE)

### 2. Cache-Then-Apply Workflow

Operations are separated into distinct phases:

1. **Analysis phase** (default): Process files with LLM, generate diffs, cache results in `.aisweeper-cache/`
2. **Application phase** (`--apply`): Read cached results and modify filesystem

This prevents accidental data loss and allows users to review changes before applying.

### 3. Cache Invalidation Strategy

Cache entries are invalidated based on:
- Git commit hash (not filesystem mtime)
- File content hash
- TTL (default: 1 hour)

Always use `git log -1 --format=%ct -- <filepath>` to determine file age, not filesystem modification time.

### 4. Safety-First Git Integration

The tool has a hard requirement for git repositories:
- Exit immediately if not in a git repo
- Display safety warnings before modification
- Check for uncommitted changes (warning only)
- Cache validation tied to git commit hash

## Component Architecture

### Processing Pipeline

```
CLI Entry → Safety Checks → Config Loader → Scanner → File Router
                                                          ↓
                                    ┌─────────────────────┴─────────────────────┐
                                    ↓                                           ↓
                         Source Processor                          Documentation Processor
                         (Clean comments)                          (Deletion analysis)
                                    ↓                                           ↓
                                    └─────────────────────┬─────────────────────┘
                                                          ↓
                                                   Cache Manager
                                                          ↓
                                                   Output Layer
                                           (Diffs / Summary / Apply)
```

### Key Components

**Scanner** (`scanner.ts`): Categorizes files as source/documentation/sacred/ephemeral. Uses git history for file age, respects `.gitignore`.

**Source Processor** (`processors/source.ts`): Generates context-aware prompts with language-specific rules and aggressiveness levels (Conservative/Moderate/Aggressive/Nuclear).

**Documentation Processor** (`processors/documentation.ts`): LLM analyzes markdown files and returns JSON with recommendation, reasoning, and confidence level. Only high-confidence DELETE recommendations are acted upon.

**Cache Manager** (`cache.ts`): Stores LLM results with git commit hash. Cache structure:
```
.aisweeper-cache/
├── manifest.json (git commit, timestamp)
├── source/
│   └── <hashed-file-path>.json
└── docs/
    └── <hashed-file-path>.json
```

**LLM Client** (`llm/client.ts`): Abstract interface supporting multiple providers (OpenAI, Anthropic, Azure, Ollama) via LiteLLM or similar library.

## Configuration System

Configuration is loaded from `.aisweeper.json` in project root and merged with defaults. Key configuration areas:

- **Aggressiveness levels**: Controls how aggressive comment removal is
- **Sacred files**: Never process these files (README.md, CONTRIBUTING.md, etc.)
- **Ephemeral patterns**: Filename patterns for auto-deletion (`*_NOTES.md`, `AI_*.md`, etc.)
- **Ephemeral age**: Days since last git commit (default: 30)
- **Deletion analysis**: Use LLM for deletion recommendations (vs pattern-only)

Environment variables can be referenced as `${OPENAI_API_KEY}`.

## Execution Modes

**Dry-run** (`--dry-run`): Fast scan without LLM calls. Estimates cost and shows what would be processed.

**Default** (no flags): Processes files with LLM, shows diffs, caches results. Does NOT modify files.

**Apply** (`--apply`): Reads from cache and applies changes. Errors if cache is missing or stale.

**Markdown-only** (`--markdown-only`): Processes only `.md` files.

## Aggressiveness Levels

When implementing prompts for the source processor:

- **Conservative**: Remove only obvious AI meta-commentary ("I'll now...", "Let me add...")
- **Moderate**: Also remove AI reasoning, stale TODOs, iteration references
- **Aggressive**: Also remove redundant comments that restate code
- **Nuclear**: Question every comment; keep only complex algorithms, security/performance notes, public API docs

## Documentation File Strategy

Files are categorized in tiers:

1. **Sacred** (never touch): README.md, CONTRIBUTING.md, LICENSE.md, CHANGELOG.md, AGENTS.md
2. **Ephemeral** (recommend deletion): Files matching patterns (`*_NOTES.md`, `AI_*.md`) AND not modified in 30+ days
3. **Analysis-based**: All other docs get LLM deletion analysis with confidence threshold

Only delete documentation if:
- Matches ephemeral pattern AND old, OR
- LLM recommends DELETE with HIGH confidence

## Error Handling Philosophy

**Critical errors** (exit immediately):
- Not a git repository
- Invalid configuration
- LLM auth failure
- Cache corruption during apply

**Warnings** (continue with confirmation):
- Uncommitted changes
- Stale cache
- Low confidence deletion recommendations

**Graceful degradation**:
- If one file fails LLM processing, continue with others
- If git history unavailable for a file, fall back to filesystem mtime

## Development Notes

### Not Yet Implemented

This is a greenfield project. Currently only ARCHITECTURE.md and ROADMAP.md exist. Refer to ROADMAP.md for implementation phases:

- **Phase 1** (Week 1-2): Project setup, CLI framework, config loader, git integration, scanner
- **Phase 2** (Week 3-4): LLM client, processors, caching, diff generation
- **Phase 3** (Week 5-6): Execution modes, reporting, error handling, testing

### Technology Decisions Still Open

- **Language**: TypeScript vs Python vs Rust (see ROADMAP.md Open Questions)
- **LLM Library**: LiteLLM (Python) vs custom TypeScript wrapper
- **Distribution**: npm vs pip vs cargo vs standalone binary

### When Implementing Processors

Source processor must:
- Include language name in prompt context
- Support all four aggressiveness levels
- Return exactly "SKIP" if no changes needed
- Generate diffs for cache storage

Documentation processor must:
- Return valid JSON: `{"recommendation": "DELETE|KEEP|PRUNE", "reasoning": "...", "confidence": "high|medium|low"}`
- Include file age (days since last commit) in analysis
- Consider filename patterns as context

### Testing Approach

- Unit tests for config loading, file categorization, diff generation, cache invalidation
- Integration tests for end-to-end flows with mock LLM responses
- Manual testing on real AI-generated codebases for validation
