# Implementation Plan: MVP - Core Comment Sweeping & Documentation Cleanup

**Branch**: `master` | **Date**: 2026-01-07 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `specs/master/spec.md`

## Summary

Build MVP of ai-comment-sweeper CLI tool that removes AI-generated meta-commentary from source code comments and analyzes markdown documentation for deletion. Implements cache-then-apply workflow with LLM-based analysis, supporting multiple LLM providers (OpenAI, Anthropic, Azure, Ollama) through abstraction layer. Enforces git repository requirement for safety, respects .gitignore, and provides dry-run/default/apply execution modes.

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: Click (CLI), LiteLLM (multi-provider LLM), GitPython (git integration), Rich (terminal output), Pydantic (config validation)
**Storage**: File-based cache in `.aisweeper-cache/` (JSON), config in `.aisweeper.json`
**Testing**: pytest, pytest-cov, integration tests with mock LLM responses
**Target Platform**: Cross-platform CLI (Linux, macOS, Windows)
**Project Type**: Single project (CLI tool)
**Performance Goals**: <60s for 100 files, <500ms startup time, <100MB memory for typical repository
**Constraints**: LLM context window limits (100K tokens for most providers), git repository required, must work offline (dry-run mode)
**Scale/Scope**: Repositories with 1-10K files, 100-1000 markdown docs, typical file size <50KB

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Pre-Phase 0 Check: ✅ PASS

All constitutional principles addressed in Technical Context and project structure.

### Post-Phase 1 Check: ✅ PASS

Design artifacts confirm full compliance:

### I. Safety First (NON-NEGOTIABLE) ✅ PASS

- ✅ **Design**: `safety.py` module implements git repository detection, verification, and safety warnings
- ✅ **Data Model**: All file operations tied to git commit hash for cache invalidation
- ✅ **Contracts**: Apply mode requires valid cache with matching commit hash
- ✅ **Implementation**: All operations reversible via git (no direct file modifications without cache)

### II. Cache-Then-Apply Workflow (NON-NEGOTIABLE) ✅ PASS

- ✅ **Design**: Separate `cache.py` and `apply.py` modules enforce separation
- ✅ **Data Model**: `CacheEntry` model stores git commit hash for validation
- ✅ **Contracts**: Default mode never modifies files, apply mode reads from cache only
- ✅ **Implementation**: Cache validation checks commit hash, content hash, and TTL

### III. Two-Track Processing Model ✅ PASS

- ✅ **Design**: Separate `SourceProcessor` and `DocumentationProcessor` in `processors/`
- ✅ **Data Model**: Distinct `SourceFile` and `DocumentationFile` models with different fields
- ✅ **Contracts**: Separate prompt templates for source cleaning vs doc deletion analysis
- ✅ **Implementation**: Sacred files enforced in scanner, four aggressiveness levels in prompts

### IV. LLM-Agnostic Architecture ✅ PASS

- ✅ **Design**: `llm/client.py` abstraction layer using LiteLLM
- ✅ **Research**: LiteLLM selected for multi-provider support with unified API
- ✅ **Data Model**: `LLMConfig` model supports provider/model configuration
- ✅ **Implementation**: No provider-specific code outside `llm/` module

### V. Explicit Configuration Over Convention ✅ PASS

- ✅ **Design**: `config.py` with Pydantic models for validation
- ✅ **Data Model**: Complete `Config` hierarchy with nested models and defaults
- ✅ **Research**: Pydantic v2 selected for environment variable resolution and validation
- ✅ **Implementation**: All behavior configurable via `.aisweeper.json`

**Constitution Gates**: ALL PASS (Pre-Phase 0 and Post-Phase 1)
**Design Compliance**: 100% - All principles implemented in design artifacts
**No violations or complexity tracking required**

## Project Structure

### Documentation (this feature)

```text
specs/master/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0 output (technology decisions)
├── data-model.md        # Phase 1 output (core data structures)
├── quickstart.md        # Phase 1 output (developer getting started)
└── contracts/           # Phase 1 output (LLM prompt templates, cache schema)
```

### Source Code (repository root)

```text
src/
└── ai_comment_sweeper/
    ├── __init__.py
    ├── cli.py                    # CLI entry point, argument parsing
    ├── safety.py                 # Git checks, safety warnings
    ├── config.py                 # Configuration loading, validation
    ├── scanner.py                # Repository traversal, file categorization
    ├── cache.py                  # Cache management, invalidation
    ├── diff.py                   # Diff generation, colorization
    ├── reporter.py               # Summary reporting, statistics
    ├── apply.py                  # File modification from cache
    ├── models/
    │   ├── __init__.py
    │   ├── config.py             # Pydantic config models
    │   ├── file.py               # SourceFile, DocumentationFile models
    │   ├── cache.py              # CacheEntry, CacheManifest models
    │   └── result.py             # ProcessingResult models
    ├── processors/
    │   ├── __init__.py
    │   ├── base.py               # Base processor interface
    │   ├── source.py             # Source file comment cleaning
    │   └── documentation.py      # Documentation deletion analysis
    └── llm/
        ├── __init__.py
        ├── client.py             # LLM abstraction layer
        ├── prompts.py            # Prompt templates
        └── provider.py           # Provider-specific adapters

tests/
├── __init__.py
├── conftest.py                   # Pytest fixtures
├── unit/
│   ├── test_config.py
│   ├── test_scanner.py
│   ├── test_cache.py
│   ├── test_diff.py
│   └── test_processors.py
├── integration/
│   ├── test_dry_run.py
│   ├── test_default_mode.py
│   ├── test_apply_mode.py
│   └── test_llm_client.py
└── fixtures/
    ├── sample_repo/              # Test git repository
    └── mock_responses/           # LLM mock responses
```

**Structure Decision**: Single project structure selected. This is a standalone CLI tool with no frontend/backend separation or mobile components. Python package structure follows standard conventions with `src/` layout for clean separation between source and tests. Core modules (cli, safety, config, scanner) at top level, domain models in `models/`, specialized processors in `processors/`, LLM integration in `llm/`.

## Complexity Tracking

> **No constitutional violations - this section is empty.**
