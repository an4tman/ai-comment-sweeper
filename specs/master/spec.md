# Feature Specification: MVP - Core Comment Sweeping & Documentation Cleanup

**Feature Branch**: `master`
**Created**: 2026-01-07
**Status**: Draft
**Input**: Initial MVP implementation based on ARCHITECTURE.md, CLAUDE.md, ROADMAP.md

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Safe Preview of Comment Cleanup (Priority: P1) 🎯 MVP

Developer wants to see what AI-generated comments would be removed from their codebase before making any changes.

**Why this priority**: Core safety principle - users must review changes before application. This is the foundation for all other functionality.

**Independent Test**: Can run dry-run mode on a sample repository and receive a summary without any files being modified.

**Acceptance Scenarios**:

1. **Given** a git repository with Python/TypeScript files containing AI meta-commentary, **When** user runs `ai-comment-sweeper --dry-run`, **Then** tool displays summary of files to process, estimated cost, and exits without modifying any files
2. **Given** not in a git repository, **When** user runs any command, **Then** tool displays error "Not a git repository" and exits immediately
3. **Given** a repository with uncommitted changes, **When** user runs tool, **Then** tool displays warning but continues (dry-run only)

---

### User Story 2 - Analyze and Cache Comment Cleanup (Priority: P1) 🎯 MVP

Developer wants to process their codebase with LLM to analyze which comments should be removed, review the diffs, then decide whether to apply.

**Why this priority**: Implements the cache-then-apply workflow (constitution principle II). Users can review all proposed changes before applying.

**Independent Test**: Run default mode on a sample repository, verify cache is created with diffs, no files are modified, and diffs are displayed.

**Acceptance Scenarios**:

1. **Given** a git repository with source files, **When** user runs `ai-comment-sweeper` (default mode), **Then** tool processes files with LLM, displays diffs, caches results in `.aisweeper-cache/`, and does NOT modify source files
2. **Given** processed files, **When** examining cache, **Then** cache contains git commit hash, file content hash, timestamp, and generated diffs
3. **Given** configuration with aggressiveness level, **When** processing source files, **Then** LLM applies corresponding removal strategy (Conservative/Moderate/Aggressive/Nuclear)

---

### User Story 3 - Apply Cached Changes (Priority: P1) 🎯 MVP

Developer has reviewed diffs from cache and wants to apply the changes to their files.

**Why this priority**: Completes the cache-then-apply workflow. Without this, the tool only analyzes but never cleans.

**Independent Test**: Run default mode to cache changes, then run --apply to modify files, verify files are changed per cached diffs.

**Acceptance Scenarios**:

1. **Given** valid cache from previous run, **When** user runs `ai-comment-sweeper --apply`, **Then** tool applies all cached changes to source files and displays summary
2. **Given** no cache or stale cache, **When** user runs `--apply`, **Then** tool errors with "Cache missing or stale, run analysis first"
3. **Given** files modified since cache creation, **When** running `--apply`, **Then** tool detects hash mismatch and errors safely

---

### User Story 4 - Markdown Documentation Analysis (Priority: P2)

Developer wants to identify and remove ephemeral AI-generated documentation files that are no longer relevant.

**Why this priority**: Two-track processing model (constitution principle III). Completes the documentation cleanup capability.

**Independent Test**: Run tool on repository with old AI_NOTES.md files, verify deletion recommendations are generated.

**Acceptance Scenarios**:

1. **Given** markdown files matching ephemeral patterns AND older than age threshold, **When** running analysis, **Then** tool recommends deletion with confidence HIGH
2. **Given** sacred files (README.md, LICENSE.md, CONTRIBUTING.md), **When** running analysis, **Then** these files are never processed or recommended for deletion
3. **Given** recent markdown files (< 30 days), **When** running LLM analysis, **Then** tool uses content analysis to determine DELETE/KEEP/PRUNE with confidence levels
4. **Given** `--markdown-only` flag, **When** running tool, **Then** only .md files are processed, source files are skipped

---

### User Story 5 - Configurable Behavior (Priority: P3)

Developer wants to customize tool behavior for their project's specific needs via configuration file.

**Why this priority**: Explicit configuration over convention (constitution principle V). Enables adaptation to different codebases.

**Independent Test**: Create custom .aisweeper.json, verify tool respects configuration settings.

**Acceptance Scenarios**:

1. **Given** `.aisweeper.json` with custom aggressiveness level, **When** processing files, **Then** tool uses specified level instead of default
2. **Given** configuration with environment variable `${OPENAI_API_KEY}`, **When** loading config, **Then** tool resolves variable from environment
3. **Given** invalid configuration file, **When** loading config, **Then** tool displays validation errors and exits
4. **Given** configuration with custom sacred files list, **When** scanning repository, **Then** tool skips processing those files

---

### Edge Cases

- What happens when LLM returns "SKIP" (no changes needed)?
- How does system handle LLM API failures mid-processing?
- What if cache TTL expires between analysis and apply?
- How does tool handle binary files or unsupported file types?
- What if git history is unavailable for a specific file (shallow clone)?
- How does tool handle files larger than LLM context window?

## Requirements *(mandatory)*

### Functional Requirements

**Safety & Git Integration:**
- **FR-001**: System MUST verify git repository exists before any operation
- **FR-002**: System MUST display safety warning banner before destructive operations
- **FR-003**: System MUST warn about uncommitted changes (warning only, not blocking)
- **FR-004**: System MUST exit immediately if not in git repository
- **FR-005**: All operations MUST be reversible through git

**Execution Modes:**
- **FR-006**: System MUST support dry-run mode (`--dry-run`) that estimates without processing
- **FR-007**: System MUST support default mode that analyzes, caches, and displays diffs without modifying files
- **FR-008**: System MUST support apply mode (`--apply`) that reads cache and modifies files
- **FR-009**: System MUST support markdown-only mode (`--markdown-only`) that processes only .md files

**Configuration:**
- **FR-010**: System MUST load configuration from `.aisweeper.json` in project root
- **FR-011**: System MUST merge user config with default configuration
- **FR-012**: System MUST validate configuration schema and error on invalid config
- **FR-013**: System MUST support environment variable resolution (e.g., `${OPENAI_API_KEY}`)
- **FR-014**: System MUST support configurable aggressiveness levels: Conservative, Moderate, Aggressive, Nuclear

**Source File Processing:**
- **FR-015**: System MUST categorize files as source (.js, .ts, .py, .go, etc.) or documentation (.md)
- **FR-016**: System MUST respect `.gitignore` patterns when scanning
- **FR-017**: System MUST generate language-aware prompts for source files
- **FR-018**: System MUST call LLM to clean AI meta-commentary from comments
- **FR-019**: System MUST generate unified diffs for all changes
- **FR-020**: LLM MUST return "SKIP" if no changes are needed

**Documentation Processing:**
- **FR-021**: System MUST never process sacred files (README.md, CONTRIBUTING.md, LICENSE.md, CHANGELOG.md, AGENTS.md)
- **FR-022**: System MUST detect ephemeral files by pattern matching and age threshold (default: 30 days)
- **FR-023**: System MUST use LLM to analyze documentation for deletion with confidence levels (DELETE/KEEP/PRUNE)
- **FR-024**: System MUST only recommend deletion for high-confidence results

**Caching:**
- **FR-025**: System MUST cache LLM results in `.aisweeper-cache/` directory
- **FR-026**: Cache entries MUST include: git commit hash, file content hash, timestamp, results
- **FR-027**: Cache invalidation MUST be based on git commit hash, NOT filesystem mtime
- **FR-028**: System MUST validate cache before apply and error if missing/stale
- **FR-029**: Cache MUST have configurable TTL (default: 1 hour)

**LLM Integration:**
- **FR-030**: System MUST support multiple LLM providers: OpenAI, Anthropic, Azure OpenAI, Ollama
- **FR-031**: System MUST abstract provider differences behind unified interface
- **FR-032**: System MUST implement retry logic for transient failures
- **FR-033**: System MUST track token usage and estimate costs
- **FR-034**: System MUST handle rate limiting gracefully

**Output & Reporting:**
- **FR-035**: System MUST display colorized unified diffs in terminal
- **FR-036**: System MUST generate summary report with: files cleaned, comments removed/kept, docs deleted, cost estimates
- **FR-037**: System MUST display deletion recommendations with reasoning and confidence
- **FR-038**: System MUST provide clear, actionable error messages

### Key Entities

- **ConfigFile**: Represents `.aisweeper.json` with aggressiveness, LLM settings, sacred files, ephemeral patterns, ignore patterns
- **SourceFile**: Source code file with path, language, content, comment count
- **DocumentationFile**: Markdown file with path, age (days since last commit), ephemeral pattern match status
- **CacheEntry**: Cached LLM result with git commit hash, file hash, timestamp, diff/recommendation
- **ProcessingResult**: Result of LLM processing (cleaned content, diff, or deletion recommendation)
- **LLMProvider**: Abstraction for OpenAI, Anthropic, Azure, Ollama with unified interface

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Tool successfully processes 10+ real AI-generated codebases without errors
- **SC-002**: Correctly identifies and removes >80% of AI meta-commentary in test cases
- **SC-003**: Zero false positives on sacred documentation files in test suite
- **SC-004**: Cache invalidation works correctly across git operations (reset, checkout, commit)
- **SC-005**: Sub-minute processing time for repositories with <100 files
- **SC-006**: Clear diffs that developers can review in <5 minutes for typical repository
- **SC-007**: Apply mode successfully modifies files matching cached diffs with 100% accuracy
- **SC-008**: Positive feedback from 5+ beta users on safety and usefulness
