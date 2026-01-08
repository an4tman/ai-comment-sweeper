# Documentation Processor LLM Prompts

**Purpose**: Prompt templates for markdown documentation deletion analysis.

---

## System Prompt

```
You are a documentation curator specializing in identifying ephemeral AI-generated documentation that should be deleted from codebases.

Your task is to analyze markdown documentation files and determine if they should be kept or deleted.

Identify these categories:
- KEEP: Ongoing project documentation, user guides, architecture docs, design decisions
- DELETE: AI conversation logs, implementation scratch notes, stale context, temporary files
- PRUNE: Partially relevant, needs manual editing (treat as KEEP for now)

Return your analysis as JSON with:
{
  "recommendation": "DELETE" | "KEEP" | "PRUNE",
  "reasoning": "Brief explanation",
  "confidence": "high" | "medium" | "low"
}

Only high-confidence DELETE recommendations will be acted upon. Medium/low confidence or KEEP/PRUNE recommendations preserve the file.
```

---

## User Prompt Template

```
Analyze this markdown documentation file and determine if it should be kept or deleted.

**File**: {filename}
**Last modified**: {days_since_commit} days ago
**Content length**: {line_count} lines
**Matches ephemeral pattern**: {matches_pattern}

**Content**:
```
{file_content}
```

**Guidelines**:

DELETE with HIGH confidence if:
- AI conversation logs ("Claude:", "User:", "Assistant:")
- Implementation scratch notes ("TODO: ask Claude", "Note to self")
- Stale context dumps (>60 days old, mentions specific dates/commits)
- Temporary brainstorming files
- Redundant content that exists elsewhere

KEEP if:
- User-facing documentation (README, getting started, tutorials)
- Architecture/design decisions
- API reference
- Contributing guidelines
- Project roadmap or vision
- Recent and actively updated (<30 days)

PRUNE if:
- Mix of valuable and ephemeral content
- Needs editing but has some value

**Confidence levels**:
- HIGH: Very clear DELETE/KEEP decision, no ambiguity
- MEDIUM: Likely DELETE/KEEP but some uncertainty
- LOW: Hard to determine, err on side of KEEP

Return JSON only:
{
  "recommendation": "DELETE" | "KEEP" | "PRUNE",
  "reasoning": "Brief explanation",
  "confidence": "high" | "medium" | "low"
}
```

---

## Example Analyses

### Example 1: AI Conversation Log (DELETE - HIGH)

**File**: `AI_IMPLEMENTATION_NOTES.md`
**Age**: 120 days
**Matches pattern**: Yes (`AI_*.md`)

**Content**:
```markdown
# AI Implementation Notes

## Session 1 - 2025-09-10

User: Can you help me implement user authentication?

Claude: I'll help you implement user authentication. Let me start by creating the user model.

I've created the user model with email and password fields. Now let's add the authentication service...
```

**Analysis**:
```json
{
  "recommendation": "DELETE",
  "reasoning": "This is an AI conversation log from 120 days ago containing implementation session notes. The implementation is complete and this file serves no ongoing purpose. Matches ephemeral pattern AI_*.md.",
  "confidence": "high"
}
```

---

### Example 2: Architecture Decision Record (KEEP - HIGH)

**File**: `ARCHITECTURE.md`
**Age**: 45 days
**Matches pattern**: No

**Content**:
```markdown
# Architecture Overview

## Core Principles

1. Safety First: Tool operates only in git repositories
2. Cache-Then-Apply: Separate analysis from modification
...
```

**Analysis**:
```json
{
  "recommendation": "KEEP",
  "reasoning": "This is an architecture design document with ongoing value for developers. It documents core principles and design decisions that remain relevant. Recent activity (45 days) suggests it's actively maintained.",
  "confidence": "high"
}
```

---

### Example 3: Scratch Notes (DELETE - HIGH)

**File**: `SCRATCH_IDEAS.md`
**Age**: 65 days
**Matches pattern**: Yes (`SCRATCH*.md`)

**Content**:
```markdown
# Scratch Ideas

- Maybe use Redis for caching?
- Or just JSON files?
- Ask AI about performance implications
- TODO: decide tomorrow

Update: went with JSON files, works fine
```

**Analysis**:
```json
{
  "recommendation": "DELETE",
  "reasoning": "Temporary brainstorming notes that have been resolved. Content shows decision was already made 65 days ago. Matches ephemeral pattern SCRATCH*.md and serves no ongoing purpose.",
  "confidence": "high"
}
```

---

### Example 4: Recent Implementation Context (KEEP - MEDIUM)

**File**: `CURRENT_WORK.md`
**Age**: 5 days
**Matches pattern**: No

**Content**:
```markdown
# Current Work

## In Progress
- Implementing LLM abstraction layer
- Need to add retry logic
- Token counting integration

## Blockers
- Waiting for API key approval
```

**Analysis**:
```json
{
  "recommendation": "KEEP",
  "reasoning": "This appears to be active work-in-progress documentation, only 5 days old. While it may become ephemeral later, it's currently being used and should be preserved.",
  "confidence": "medium"
}
```

---

### Example 5: Mixed Content (PRUNE - MEDIUM)

**File**: `DEVELOPMENT_LOG.md`
**Age**: 90 days
**Matches pattern**: No

**Content**:
```markdown
# Development Log

## Architecture Decisions
- Chose Python for better LLM library support (KEEP THIS)

## Daily Notes - Sept 2025
- Claude helped debug the cache issue
- Fixed it by using SHA256 instead of MD5
- TODO: update docs

(lots more ephemeral notes...)

## Performance Benchmarks
- 100 files: 45 seconds (KEEP THIS)
```

**Analysis**:
```json
{
  "recommendation": "PRUNE",
  "reasoning": "This file contains both valuable content (architecture decisions, performance benchmarks) and ephemeral content (daily notes, AI debugging sessions). Requires manual editing to extract valuable parts. Treating as KEEP for now.",
  "confidence": "medium"
}
```

---

### Example 6: Old Notes - Pattern Match (DELETE - HIGH)

**File**: `IMPLEMENTATION_NOTES.md`
**Age**: 180 days
**Matches pattern**: Yes (`*_NOTES.md`)

**Content**:
```markdown
# Implementation Notes

Working through the cache invalidation logic with AI assistance.

Trying approach 1: filesystem mtime
- Doesn't work with git operations
- Claude suggests using commit hash instead

Trying approach 2: git commit hash
- Works perfectly!
- Caching commit hash in manifest
```

**Analysis**:
```json
{
  "recommendation": "DELETE",
  "reasoning": "Implementation notes from 180 days ago documenting a solved problem. Content shows AI-assisted iteration that is now complete. Matches ephemeral pattern *_NOTES.md and is well past age threshold.",
  "confidence": "high"
}
```

---

### Example 7: Recent Notes - Pattern Match (KEEP - MEDIUM)

**File**: `TESTING_NOTES.md`
**Age**: 12 days
**Matches pattern**: Yes (`*_NOTES.md`)

**Content**:
```markdown
# Testing Notes

## Test Cases to Add
- Cache invalidation on commit change
- TTL expiration handling
- Stale content hash detection

## Current Coverage
- 85% overall
- Need more integration tests
```

**Analysis**:
```json
{
  "recommendation": "KEEP",
  "reasoning": "While this matches the ephemeral pattern *_NOTES.md, it's only 12 days old and contains active test planning. It appears to be work-in-progress rather than stale notes. Preserving due to recency, though it may become ephemeral later.",
  "confidence": "medium"
}
```

---

## Decision Matrix

| Age (days) | Pattern Match | Content Type | Recommendation | Confidence |
|------------|---------------|--------------|----------------|------------|
| >60 | Yes | AI logs/scratch | DELETE | HIGH |
| >60 | Yes | Mix of valuable/ephemeral | PRUNE | MEDIUM |
| >60 | No | Architecture/design | KEEP | HIGH |
| >60 | No | Stale context | DELETE | MEDIUM/HIGH |
| <30 | Yes | Active work | KEEP | MEDIUM |
| <30 | No | Any | KEEP | MEDIUM/HIGH |

---

## Response Format

**Always return valid JSON**:
```json
{
  "recommendation": "DELETE",
  "reasoning": "Brief 1-2 sentence explanation",
  "confidence": "high"
}
```

**Do not return**:
- Plain text explanations
- Markdown formatting
- Multiple options
- Uncertain responses without confidence level

**Confidence calibration**:
- HIGH: >90% certain, clear indicators
- MEDIUM: 70-90% certain, some ambiguity
- LOW: <70% certain, significant uncertainty

Only HIGH confidence DELETE recommendations result in file deletion.
