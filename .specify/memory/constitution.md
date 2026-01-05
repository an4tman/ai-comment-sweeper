<!--
SYNC IMPACT REPORT
==================
Version change: [UNVERSIONED] → 1.0.0
Modified principles: N/A (initial version)
Added sections:
  - Core Principles (5 principles)
  - Security & Safety Standards
  - Development Workflow
  - Governance
Removed sections: N/A (initial version)
Templates requiring updates:
  ✅ plan-template.md - reviewed, Constitution Check section aligns
  ✅ spec-template.md - reviewed, requirement patterns align
  ✅ tasks-template.md - reviewed, task categorization aligns
Follow-up TODOs: None
==================
-->

# ai-comment-sweeper Constitution

## Core Principles

### I. Safety First (NON-NEGOTIABLE)

**The tool MUST operate exclusively in git repositories with comprehensive safety checks.**

- Git repository detection is a hard requirement; exit immediately if not in a git repo
- Display safety warning banners before any destructive operations
- Warn users about uncommitted changes (warning only, not blocking)
- Never modify files without explicit user acknowledgment (via `--apply` flag)
- All operations must be reversible through git

**Rationale**: This tool performs destructive operations (file modifications, deletions). Without git version control, users have no recovery mechanism. Safety checks prevent catastrophic data loss.

### II. Cache-Then-Apply Workflow (NON-NEGOTIABLE)

**Analysis and modification MUST be separated into distinct phases.**

- Default behavior: analyze with LLM, generate diffs, cache results—do NOT modify files
- Application phase (`--apply`): read cached results and modify filesystem
- Cache invalidation tied to git commit hash, NOT filesystem modification time
- Cache entries include: git commit hash, file content hash, timestamp, TTL (default: 1 hour)
- Apply mode MUST error if cache is missing, stale, or invalid

**Rationale**: Separating analysis from modification prevents accidental data loss and allows users to review all proposed changes before application. Git-based cache invalidation ensures cache validity across git operations (reset, checkout, etc.) that filesystem timestamps cannot capture.

### III. Two-Track Processing Model

**Source files and documentation files require fundamentally different processing strategies.**

- **Source files** (`.js`, `.ts`, `.py`, etc.): LLM cleans AI meta-commentary while preserving meaningful comments
- **Documentation files** (`.md`): LLM analyzes for deletion recommendation with confidence levels (DELETE/KEEP/PRUNE)
- Sacred files (README.md, CONTRIBUTING.md, LICENSE.md, CHANGELOG.md, AGENTS.md) MUST never be processed
- Ephemeral documentation deletion requires BOTH pattern match AND age threshold OR high-confidence LLM recommendation
- Source processor MUST support four aggressiveness levels: Conservative, Moderate, Aggressive, Nuclear

**Rationale**: Comments in source code and standalone documentation serve different purposes and have different lifespans. Source code comments often need curation, not deletion. Documentation files may become entirely obsolete and should be removed. Different strategies prevent inappropriate operations.

### IV. LLM-Agnostic Architecture

**The system MUST support multiple LLM providers through a unified abstraction layer.**

- Abstract LLM provider differences behind a common interface
- Support OpenAI, Anthropic, Azure OpenAI, and local models (Ollama)
- Abstraction layer handles: API calls, retries, rate limiting, token counting, cost estimation
- Users specify provider and model in `.aisweeper.json` configuration
- No provider-specific logic outside the abstraction layer

**Rationale**: LLM provider landscape is rapidly evolving. Users have different preferences, cost constraints, and privacy requirements. Provider-agnostic design ensures longevity and flexibility.

### V. Explicit Configuration Over Convention

**All behavior-altering settings MUST be configurable via `.aisweeper.json` with sensible defaults.**

- Configuration file location: `.aisweeper.json` in project root
- Schema validation required; invalid configs MUST error with clear messages
- Support environment variable resolution (e.g., `${OPENAI_API_KEY}`)
- Default configuration provides working baseline for common use cases
- Sacred files, ephemeral patterns, aggressiveness levels, LLM settings, and ignore patterns MUST all be configurable

**Rationale**: Different projects have different comment styles, documentation practices, and quality standards. Hard-coded behavior limits tool applicability. Explicit configuration enables adaptation to diverse codebases while defaults enable quick adoption.

## Security & Safety Standards

### Data Protection

- **API Key Management**: MUST support environment variable references; NEVER log or display API keys
- **Code Injection Prevention**: MUST sanitize file contents before sending to LLM (escape injection patterns)
- **File System Access**: MUST respect `.gitignore` and configured ignore patterns
- **Deletion Safety**: ONLY delete documentation files with high-confidence LLM recommendations OR explicit ephemeral pattern + age match

### Error Handling Philosophy

**Critical errors** (exit immediately):
- Not a git repository
- Invalid configuration file
- LLM API authentication failure
- Cache corruption during apply

**Warnings** (continue with user confirmation):
- Uncommitted changes detected
- Cache older than TTL
- Low confidence deletion recommendations

**Graceful degradation**:
- If LLM call fails for one file, continue with others
- If git history unavailable for a file, fall back to filesystem mtime
- If cache is partially stale, regenerate only affected files

## Development Workflow

### Implementation Workflow

1. **Requirements**: Feature specifications in `spec.md` with user stories and acceptance criteria
2. **Planning**: Implementation plan in `plan.md` with technical context and structure
3. **Task Breakdown**: Concrete tasks in `tasks.md` organized by user story priority
4. **Implementation**: Follow phases sequentially; complete foundational work before user stories
5. **Testing**: Unit tests for core logic; integration tests for end-to-end flows; manual validation on real codebases

### Testing Requirements

**Unit Tests** (required for):
- Configuration loading and validation
- File categorization logic
- Diff generation
- Cache invalidation logic

**Integration Tests** (required for):
- End-to-end dry-run mode
- End-to-end default + apply mode
- LLM client with mock responses
- Git integration

**Manual Testing** (required for):
- Real codebases with AI-generated comments
- Safety checks effectiveness
- Diff accuracy
- Deletion recommendation quality

### Code Quality Standards

- **Language**: TypeScript for type safety and ecosystem compatibility (per ROADMAP.md open questions)
- **Linting**: ESLint and Prettier for consistent formatting
- **Comments**: Follow aggressiveness philosophy—comment non-obvious logic, algorithms, security/performance considerations; avoid restating code
- **Modularity**: Clear separation of concerns: CLI → Safety → Config → Scanner → Processors → Cache → Output
- **Error Messages**: Clear, actionable error messages that guide users to resolution

## Governance

### Constitution Supremacy

This constitution supersedes all other practices, documentation, and conventions. When conflicts arise, constitution principles take precedence.

### Amendment Process

1. **Proposal**: Document proposed change with rationale
2. **Version Bump**: Determine semantic version increment:
   - **MAJOR**: Backward incompatible principle removals or redefinitions
   - **MINOR**: New principles or materially expanded guidance
   - **PATCH**: Clarifications, wording, typo fixes
3. **Sync Impact**: Update all dependent templates (plan-template.md, spec-template.md, tasks-template.md)
4. **Approval**: Document approval and update `Last Amended` date

### Compliance Verification

- All implementation plans MUST include "Constitution Check" section (see plan-template.md)
- Code reviews MUST verify compliance with core principles
- Violations of NON-NEGOTIABLE principles are blocking issues
- Complexity or architectural deviations MUST be justified in plan.md "Complexity Tracking" section

### Runtime Guidance

Development guidance for AI agents (Claude) is maintained in `CLAUDE.md`. This file provides implementation context, component architecture, and development notes that complement constitutional principles.

**Version**: 1.0.0 | **Ratified**: 2026-01-05 | **Last Amended**: 2026-01-05
