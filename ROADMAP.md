# ai-comment-sweeper Roadmap

## Project Vision

Create a CLI tool that removes AI-generated meta-commentary, stale comments, and ephemeral documentation from codebases developed with AI assistance, helping developers maintain clean, production-ready code.

## Release Strategy

- **v0.1.0**: MVP - Core functionality for comment cleaning and basic markdown handling
- **v0.2.0**: Enhanced markdown intelligence and quality-of-life improvements
- **v0.3.0**: Advanced features and optimizations
- **v1.0.0**: Production-ready with comprehensive testing and documentation

---

## MVP (v0.1.0) - Core Functionality

### Essential Features

#### 1. Execution Modes ✅ PRIORITY
- [ ] **Dry-run mode** (`--dry-run`)
  - Fast scan without LLM calls
  - Count comments via regex
  - Estimate token count and cost
  - List ephemeral docs detected

- [ ] **Default mode** (no flags)
  - Process files with LLM
  - Generate and display diffs
  - Cache results in `.aisweeper-cache/`
  - Show deletion recommendations for docs

- [ ] **Apply mode** (`--apply`)
  - Read from cache (error if cache missing/stale)
  - Apply all changes to filesystem
  - Delete recommended docs
  - Generate summary report

#### 2. Safety & Git Integration ✅ PRIORITY
- [ ] Hard requirement: Must be in git repository
- [ ] Display safety warning banner
- [ ] Check for uncommitted changes (warning, not blocking)
- [ ] Exit with helpful error message if not in git repo

#### 3. Configuration System ✅ PRIORITY
- [ ] Load `.aisweeper.json` from project root
- [ ] Merge with default configuration
- [ ] Schema validation
- [ ] Environment variable resolution (e.g., `${OPENAI_API_KEY}`)
- [ ] Support for:
  - Aggressiveness levels
  - LLM provider and model
  - Sacred files list
  - Ephemeral patterns
  - Ignore patterns

#### 4. Repository Scanner ✅ PRIORITY
- [ ] Traverse directory tree
- [ ] Respect `.gitignore`
- [ ] Categorize files:
  - Source files (`.js`, `.ts`, `.py`, `.go`, etc.)
  - Documentation files (`.md`)
  - Sacred files (skip processing)
- [ ] Check git history for file last modified date
- [ ] Flag ephemeral docs based on:
  - Age (relative to git commit)
  - Filename patterns

#### 5. Source File Processing ✅ PRIORITY
- [ ] Generate context-aware prompts
- [ ] Include language-specific context
- [ ] Support aggressiveness levels:
  - Conservative
  - Moderate
  - Aggressive
  - Nuclear
- [ ] Call LLM to clean comments
- [ ] Parse response (cleaned content or SKIP)
- [ ] Generate diffs

#### 6. Documentation File Processing ✅ PRIORITY
- [ ] Sacred files whitelist (never process)
  - `README.md`, `CONTRIBUTING.md`, `LICENSE.md`, `CHANGELOG.md`, etc.
- [ ] Ephemeral file detection heuristics:
  - Age threshold (default: 30 days since last commit)
  - Filename patterns (`*_NOTES.md`, `AI_*.md`, etc.)
- [ ] LLM-based deletion analysis
  - Recommendation: DELETE, KEEP, or PRUNE
  - Reasoning
  - Confidence level (high/medium/low)
- [ ] **Confidence threshold**: Only recommend deletion for high-confidence results

#### 7. LLM Integration ✅ PRIORITY
- [ ] LLM abstraction layer
- [ ] Use library for multi-provider support (LiteLLM or similar)
- [ ] Support providers:
  - OpenAI
  - Anthropic
  - Azure OpenAI
  - Local models (Ollama)
- [ ] Token counting
- [ ] Cost estimation
- [ ] Basic retry logic

#### 8. Caching System ✅ PRIORITY
- [ ] Store LLM results in `.aisweeper-cache/`
- [ ] Cache structure:
  ```
  .aisweeper-cache/
  ├── manifest.json
  ├── source/
  └── docs/
  ```
- [ ] **Cache invalidation tied to git commit hash**
- [ ] Validate cache before apply
- [ ] TTL support (default: 1 hour)

#### 9. Diff Display ✅ PRIORITY
- [ ] Generate unified diffs
- [ ] Colorize output for terminal
- [ ] Show file paths
- [ ] Display deletion recommendations with reasoning

#### 10. Summary Report ✅ PRIORITY
- [ ] Count of files cleaned
- [ ] Comments removed vs kept
- [ ] Ephemeral docs deleted
- [ ] Token/cost savings estimate
- [ ] Git status summary
- [ ] Next steps suggestions

#### 11. Markdown-Only Mode ✅ PRIORITY
- [ ] `--markdown-only` flag
- [ ] Process only `.md` files
- [ ] Useful for separate doc cleanup passes

---

## v0.2.0 - Enhanced Intelligence

### High-Priority Backlog

#### 1. Advanced Markdown Detection
- [ ] **Date-stamped file detection**
  - `NOTES_2024-01-15.md` (older than X days → delete)
  - `MEETING_2024_Q1.md`
  - Configurable date formats

- [ ] **Content-based detection**
  - Analyze file contents, not just filename
  - Detect files with mostly code snippets + AI meta-commentary
  - Detect files referencing specific git commits/dates

- [ ] **AI collaboration metadata**
  - Detect `.ai/` directories
  - Common AI tool output files

#### 2. Language-Aware Processing
- [ ] Different handling for:
  - Docstrings (Python, JS)
  - JSDoc comments
  - XML/HTML comments
  - Inline vs block comments
- [ ] Language-specific aggressiveness rules
- [ ] Preserve important language conventions (e.g., Python docstrings for public APIs)

#### 3. Whitelist/Blacklist Patterns
- [ ] User-defined comment patterns to always keep
  - Regex patterns: `TODO\(john\):`, `SECURITY:`, `PERF:`
- [ ] User-defined comment patterns to always remove
  - `// Claude:`, `// AI Note:`, `// GPT:`
- [ ] File-level overrides in config

---

## v0.3.0 - Quality of Life & Optimization

### Medium-Priority Backlog

#### 1. Interactive Mode
- [ ] `--interactive` flag
- [ ] Show each diff individually
- [ ] Prompt: "Apply? [y/n/q]"
- [ ] Allow selective application

#### 2. Partial Application
- [ ] Support path-based partial application
  - `ai-comment-sweeper src/components/ --apply`
- [ ] Apply only to specific subdirectories
- [ ] Useful for large codebases

#### 3. Cost Optimization
- [ ] **Parallel LLM calls** with concurrency limit
- [ ] **File size-based model selection**
  - Small files → cheaper model
  - Large files → more capable model
- [ ] **Skip files with no comments** (regex pre-check)
- [ ] Streaming responses for large files
- [ ] Incremental processing (only changed files)

#### 4. Enhanced Reporting
- [ ] Export report to JSON/markdown
- [ ] Detailed breakdown by file type
- [ ] Cost breakdown by operation
- [ ] Time saved estimates
- [ ] Historical comparison (if run multiple times)

#### 5. Plugin System
- [ ] Custom processors for new file types
- [ ] Custom LLM providers
- [ ] Pre/post processing hooks
- [ ] User-defined deletion rules

---

## Future Considerations (No Version Assigned)

### Low-Priority Backlog

#### 1. IDE Integration
- [ ] VS Code extension
- [ ] JetBrains plugin
- [ ] In-editor diff preview

#### 2. CI/CD Integration
- [ ] GitHub Action
- [ ] GitLab CI template
- [ ] Pre-commit hook
- [ ] Automated PR comments

#### 3. Advanced Analytics
- [ ] Track comment cleanliness over time
- [ ] Detect AI comment patterns
- [ ] Suggest custom rules based on repo history
- [ ] ML-based comment classification (avoid LLM costs)

#### 4. Collaborative Features
- [ ] Share configurations across team
- [ ] Centralized sacred files registry
- [ ] Team-wide ephemeral patterns

#### 5. Multi-Repository Support
- [ ] Process entire organizations
- [ ] Monorepo support
- [ ] Cross-repo pattern detection

---

## Implementation Order

### Phase 1: Foundation (Week 1-2)
1. Project setup (TypeScript, build system)
2. CLI framework and argument parsing
3. Configuration loader
4. Git integration and safety checks
5. Repository scanner

### Phase 2: Core Processing (Week 3-4)
1. LLM abstraction layer
2. Source file processor
3. Documentation file processor
4. Caching system
5. Diff generator

### Phase 3: User Experience (Week 5-6)
1. Execution modes (dry-run, default, apply)
2. Summary reporter
3. Error handling and logging
4. Testing and bug fixes
5. Documentation and examples

### Phase 4: MVP Release (Week 7)
1. Final testing on real codebases
2. README and getting started guide
3. Publish to npm/pip/cargo
4. Gather user feedback

---

## Success Metrics

### MVP Success Criteria
- Successfully processes 10+ real codebases
- Correctly identifies and removes >80% of AI meta-commentary
- No false positives on sacred documentation
- Clear, actionable diffs
- Sub-minute processing time for <100 files
- Positive feedback from 5+ beta users

### Long-term Goals
- 1000+ GitHub stars
- 100+ production users
- Integration with major AI coding tools
- Become the standard for AI codebase cleanup

---

## Dependencies

### Core Libraries
- **CLI Framework**: Commander.js or Yargs
- **LLM Abstraction**: LiteLLM (Python) or custom TypeScript wrapper
- **Diff Generation**: diff or jsdiff
- **Git Integration**: simple-git
- **Configuration**: cosmiconfig
- **Terminal Output**: chalk, ora (spinners)

### Development
- **Testing**: Jest or Vitest
- **Linting**: ESLint / Prettier
- **TypeScript**: tsc
- **Build**: esbuild or rollup

---

## Open Questions

### Technical Decisions
- [ ] Language choice: TypeScript (Node.js) vs Python vs Rust?
  - TypeScript: Fast iteration, good ecosystem
  - Python: Better LLM libraries, easier LiteLLM integration
  - Rust: Performance, single binary distribution

- [ ] Distribution method: npm vs pip vs cargo vs standalone binary?

- [ ] LLM library choice: LiteLLM vs custom abstraction?

### Product Decisions
- [ ] Default aggressiveness level?
- [ ] Default ephemeral age threshold (30 days vs 60 days)?
- [ ] Should dry-run be the default mode (safer) or diff mode?
- [ ] Pricing: Free open-source vs premium features?

---

## Community & Contribution

### Documentation Needed
- [ ] README with quick start
- [ ] Configuration reference
- [ ] Aggressiveness level guide
- [ ] Examples for common scenarios
- [ ] Troubleshooting guide
- [ ] Contributing guidelines

### Community Building
- [ ] GitHub Discussions for Q&A
- [ ] Discord or Slack community
- [ ] Blog posts / tutorials
- [ ] Video walkthrough
- [ ] Integration examples

---

## Version History

| Version | Release Date | Highlights |
|---------|--------------|------------|
| v0.1.0  | TBD         | MVP - Core comment cleaning and markdown handling |
| v0.2.0  | TBD         | Enhanced markdown intelligence, language awareness |
| v0.3.0  | TBD         | Interactive mode, optimizations, plugins |
| v1.0.0  | TBD         | Production-ready, comprehensive testing |

---

## Notes

- This roadmap is a living document and will be updated based on user feedback
- Feature priorities may shift based on community needs
- MVP scope may be adjusted to ensure timely delivery
- Performance optimizations are intentionally deferred until after MVP to validate the concept
