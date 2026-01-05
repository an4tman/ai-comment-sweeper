# ai-comment-sweeper Architecture

## Overview

`ai-comment-sweeper` is a CLI tool that removes AI-generated meta-commentary, stale comments, and ephemeral documentation from codebases that have been developed with AI assistance.

## Core Principles

1. **Safety First**: Only operates in git repositories with clear warnings
2. **Cache-Then-Apply**: Separates analysis from modification
3. **Two-Track Processing**: Source files get comment cleaning, documentation files get deletion analysis
4. **LLM-Agnostic**: Supports multiple LLM providers via abstraction layer

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         CLI Entry                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   --dry-run  │  │   (default)  │  │    --apply   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Safety & Git Checks                       │
│  - Verify git repository exists                              │
│  - Warn about uncommitted changes                            │
│  - Display safety banner                                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Configuration Loader                       │
│  - Load .aisweeper.json                                      │
│  - Merge with defaults                                       │
│  - Validate settings                                         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Repository Scanner                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  1. Discover all files (respecting .gitignore)       │   │
│  │  2. Categorize files:                                │   │
│  │     - Source files (.js, .py, .ts, etc.)            │   │
│  │     - Documentation files (.md)                      │   │
│  │     - Sacred files (README.md, etc.)                │   │
│  │  3. Check git history for file age                  │   │
│  │  4. Apply ephemeral patterns to docs                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    ┌─────────────────┐
                    │  File Routing   │
                    └─────────────────┘
                     ↓               ↓
        ┌────────────────────┐  ┌──────────────────────┐
        │  Source Processor  │  │  Documentation       │
        │                    │  │  Processor           │
        └────────────────────┘  └──────────────────────┘
                     ↓                      ↓
        ┌────────────────────┐  ┌──────────────────────┐
        │  LLM: Clean        │  │  LLM: Analyze for    │
        │  Comments          │  │  Deletion            │
        └────────────────────┘  └──────────────────────┘
                     ↓                      ↓
                    ┌─────────────────┐
                    │  Cache Results  │
                    │  (.aisweeper-   │
                    │   cache/)       │
                    └─────────────────┘
                              ↓
                    ┌─────────────────┐
                    │  Output Layer   │
                    └─────────────────┘
                     ↓               ↓
        ┌────────────────────┐  ┌──────────────────────┐
        │  Dry-run: Summary  │  │  Default: Show Diffs │
        └────────────────────┘  └──────────────────────┘
                                         ↓
                              ┌──────────────────────┐
                              │  Apply: Modify Files │
                              │  & Delete Docs       │
                              └──────────────────────┘
                                         ↓
                              ┌──────────────────────┐
                              │  Summary Report      │
                              └──────────────────────┘
```

## Component Breakdown

### 1. CLI Entry (`cli.ts`)

**Responsibilities:**
- Parse command-line arguments
- Route to appropriate execution mode
- Handle global flags (--help, --version)

**Modes:**
- `--dry-run`: Fast scan, no LLM calls, cost estimation
- Default: Process files, generate diffs, cache results
- `--apply`: Apply cached results to filesystem
- `--markdown-only`: Process only .md files

### 2. Safety & Git Checks (`safety.ts`)

**Responsibilities:**
- Verify git repository exists (hard requirement)
- Check for uncommitted changes (warning)
- Display safety banner
- Prevent execution outside git repos

**Output:**
```
⚠️  WARNING: This tool modifies and deletes files!
⚠️  Only use in a version-controlled repository.
⚠️  Commit your work before running with --apply.
```

### 3. Configuration Loader (`config.ts`)

**Responsibilities:**
- Load `.aisweeper.json` from project root
- Merge with default configuration
- Validate schema
- Resolve environment variables

**Default Configuration:**
```json
{
  "aggressiveness": "moderate",
  "llm": {
    "provider": "anthropic",
    "model": "claude-3-5-haiku-20241022"
  },
  "markdown": {
    "sacred": ["README.md", "CONTRIBUTING.md", "LICENSE.md", "CHANGELOG.md"],
    "ephemeralPatterns": ["*_NOTES.md", "AI_*.md", "SCRATCH*.md", "TEMP*.md"],
    "ephemeralAgeDays": 30,
    "docsFolder": {
      "path": "docs/",
      "aggressiveness": "conservative"
    },
    "deletionAnalysis": true
  },
  "ignore": ["node_modules/", "vendor/", ".git/"],
  "cache": {
    "path": ".aisweeper-cache/",
    "ttl": 3600
  }
}
```

### 4. Repository Scanner (`scanner.ts`)

**Responsibilities:**
- Traverse directory tree (respecting .gitignore)
- Categorize files by type
- Check git commit history for file age
- Mark files as sacred/ephemeral/processable

**Output:**
```typescript
interface ScanResult {
  sourceFiles: SourceFile[];
  documentationFiles: DocumentationFile[];
  sacredFiles: string[];
  ephemeralFiles: EphemeralFile[];
}

interface SourceFile {
  path: string;
  language: string;
  commentCount: number; // Pre-scan estimate
}

interface DocumentationFile {
  path: string;
  daysSinceLastCommit: number;
  matchesEphemeralPattern: boolean;
}

interface EphemeralFile {
  path: string;
  reason: string;
  daysSinceLastCommit: number;
}
```

### 5. Source Processor (`processors/source.ts`)

**Responsibilities:**
- Generate context-aware prompts for source files
- Call LLM to clean comments
- Parse LLM response (cleaned code or SKIP)
- Generate diffs

**Prompt Template:**
```
You are a code comment curator. Review this {language} file and remove:

1. AI meta-commentary about the coding process
   - "I'll now...", "Let me add...", "Note: I updated this..."
2. References to previous code iterations
   - "This was changed from...", "Previously we had..."
3. Explanations of changes that were made
   - "Updated to fix bug X", "Refactored to improve Y"
4. {aggressiveness-level-specific-rules}

Preserve:
1. Documentation of function/class behavior
2. Non-obvious implementation details
3. UX/design rationale
4. Security, performance, or edge-case warnings

Aggressiveness: {level}

Return the cleaned file content. If no changes are needed, return exactly: SKIP
```

**Aggressiveness Rules:**

| Level | Remove | Keep |
|-------|--------|------|
| Conservative | Only obvious AI meta-commentary | Everything else |
| Moderate | AI reasoning, stale TODOs, iteration notes | Implementation notes, warnings, edge cases |
| Aggressive | Above + redundant comments restating code | Only non-obvious information |
| Nuclear | Above + question every comment | Only complex algorithms, security, performance, public API docs |

### 6. Documentation Processor (`processors/documentation.ts`)

**Responsibilities:**
- Analyze markdown files for relevance
- Call LLM to recommend deletion
- Output recommendation with confidence level

**Prompt Template:**
```
Analyze this markdown documentation file and determine if it should be kept or deleted.

File: {filename}
Last modified: {daysSinceLastCommit} days ago
Content length: {lineCount} lines

Identify if this is:
- KEEP: Ongoing project documentation, user guides, architecture docs
- DELETE: AI conversation logs, implementation notes, scratch files, stale context
- PRUNE: Partially relevant, needs editing (treat as KEEP for now)

Return JSON:
{
  "recommendation": "DELETE" | "KEEP" | "PRUNE",
  "reasoning": "Brief explanation",
  "confidence": "high" | "medium" | "low"
}
```

### 7. LLM Abstraction Layer (`llm/client.ts`)

**Responsibilities:**
- Abstract LLM provider differences
- Use library like LiteLLM for multi-provider support
- Handle retries and rate limiting
- Track token usage and costs

**Interface:**
```typescript
interface LLMClient {
  complete(prompt: string, systemPrompt: string): Promise<string>;
  getTokenCount(text: string): number;
  estimateCost(tokenCount: number): number;
}
```

**Supported Providers (via LiteLLM or similar):**
- OpenAI
- Anthropic
- Azure OpenAI
- Local models (Ollama, etc.)

### 8. Cache Manager (`cache.ts`)

**Responsibilities:**
- Store LLM results to avoid re-processing
- Invalidate cache based on git commit hash
- Manage cache TTL
- Provide cache statistics

**Cache Structure:**
```
.aisweeper-cache/
├── manifest.json          # Git commit hash, timestamp
├── source/
│   ├── src_file1.json
│   └── src_file2.json
└── docs/
    ├── doc1.json
    └── doc2.json
```

**Cache Entry:**
```json
{
  "filePath": "src/components/Button.tsx",
  "originalHash": "abc123...",
  "timestamp": 1704459600,
  "gitCommit": "def456...",
  "result": {
    "type": "source",
    "action": "clean",
    "originalContent": "...",
    "cleanedContent": "...",
    "diff": "..."
  }
}
```

**Invalidation Logic:**
```typescript
function isCacheValid(entry: CacheEntry): boolean {
  const currentCommit = execSync('git rev-parse HEAD').toString().trim();
  const fileHash = getFileHash(entry.filePath);
  const ageInSeconds = Date.now() / 1000 - entry.timestamp;

  return (
    entry.gitCommit === currentCommit &&
    entry.originalHash === fileHash &&
    ageInSeconds < config.cache.ttl
  );
}
```

### 9. Diff Generator (`diff.ts`)

**Responsibilities:**
- Generate unified diffs between original and cleaned files
- Colorize output for terminal display
- Provide file deletion previews

**Output Format:**
```diff
--- a/src/components/Button.tsx
+++ b/src/components/Button.tsx
@@ -10,8 +10,6 @@
 export function Button({ onClick, children }: ButtonProps) {
-  // I've added a click handler here to manage the button state
-  // Note: This was updated from the previous implementation
   const handleClick = () => {
     onClick();
   };
```

### 10. File Applicator (`apply.ts`)

**Responsibilities:**
- Read cached results
- Apply changes to source files
- Delete recommended documentation files
- Verify all operations succeed

**Safety Checks:**
- Verify cache exists and is valid
- Confirm git repository is clean (optional warning)
- Atomic operations where possible

### 11. Reporter (`reporter.ts`)

**Responsibilities:**
- Generate summary statistics
- Display before/after metrics
- Show cost estimates
- Provide next-step recommendations

**Output:**
```
✓ Cleaned 23 source files
  - Removed 147 comments
  - Kept 89 meaningful comments
  - Reduced comment bloat by 62%

✓ Deleted 4 ephemeral markdown files:
  - docs/AI_NOTES.md (180 days old)
  - IMPLEMENTATION_LOG.md (120 days old)
  - SCRATCH_IDEAS.md (45 days old)
  - CLAUDE_CONTEXT.md (67 days old)

✓ Kept 8 documentation files (recent or sacred)

Estimated context saved: ~3,200 tokens
Time saved in future AI sessions: ~$0.15 per conversation

Git status: 27 files changed
Next steps:
  git diff          # Review changes
  git add -A        # Stage if satisfied
  git commit -m "Clean AI-generated comments"
```

## Data Flow

### Dry-run Mode
```
CLI → Safety Check → Config Load → Scanner → Cost Estimator → Summary
```

### Default Mode (Generate Diffs)
```
CLI → Safety Check → Config Load → Scanner → Process Files → Cache → Show Diffs
                                                ↓
                                    ┌───────────┴───────────┐
                                    ↓                       ↓
                            Source Processor      Documentation Processor
                                    ↓                       ↓
                                LLM Client              LLM Client
```

### Apply Mode
```
CLI → Safety Check → Config Load → Cache Validator → Apply Changes → Report
```

## Error Handling

### Critical Errors (Exit immediately)
- Not a git repository
- Invalid configuration file
- LLM API authentication failure
- Cache corruption during apply

### Warnings (Continue with user confirmation)
- Uncommitted changes detected
- Cache older than TTL
- Some files failed processing
- Low confidence deletion recommendations

### Graceful Degradation
- If LLM call fails for one file, continue with others
- If cache is stale, regenerate only affected files
- If git history unavailable for a file, use filesystem mtime as fallback

## Security Considerations

1. **API Key Management**: Never log or display API keys
2. **File System Access**: Respect .gitignore and configured ignore patterns
3. **Code Injection**: Sanitize file contents before sending to LLM
4. **Deletion Safety**: Only delete files explicitly recommended by LLM with high confidence

## Performance Optimizations

### Current (MVP)
- Sequential file processing
- Cache results to avoid re-processing
- Skip files with no comments (regex pre-check)

### Future (Backlog)
- Parallel LLM calls with concurrency limit
- File size-based model selection (smaller model for simple files)
- Streaming LLM responses for large files
- Incremental processing (only changed files since last run)

## Testing Strategy

### Unit Tests
- Configuration loader and validation
- File categorization logic
- Diff generation
- Cache invalidation logic

### Integration Tests
- End-to-end dry-run mode
- End-to-end default + apply mode
- LLM client with mock responses
- Git integration

### Manual Testing
- Test on real codebases with AI-generated comments
- Verify safety checks work
- Confirm diffs are accurate
- Validate deletion recommendations

## Extensibility Points

1. **Custom Processors**: Add support for new file types
2. **Custom LLM Providers**: Plugin architecture for providers
3. **Custom Rules**: User-defined comment patterns to keep/remove
4. **Hooks**: Pre/post processing hooks for custom logic
