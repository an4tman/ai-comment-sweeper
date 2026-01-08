# Tasks: MVP - Core Comment Sweeping & Documentation Cleanup

**Input**: Design documents from `specs/master/`
**Prerequisites**: plan.md (required), spec.md (required), data-model.md, contracts/, research.md, quickstart.md

**Note**: Tests are NOT included per spec.md - no test requirements specified. Tasks focus on implementation only.

**Organization**: Tasks are grouped by user story (US1-US5) to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Single project**: `src/ai_comment_sweeper/`, `tests/` at repository root
- All tasks reference absolute module paths from src/

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create src/ai_comment_sweeper/ package directory structure
- [ ] T002 [P] Create src/ai_comment_sweeper/__init__.py with version info
- [ ] T003 [P] Create src/ai_comment_sweeper/models/__init__.py package
- [ ] T004 [P] Create src/ai_comment_sweeper/processors/__init__.py package
- [ ] T005 [P] Create src/ai_comment_sweeper/llm/__init__.py package
- [ ] T006 [P] Create tests/__init__.py and tests/conftest.py with pytest fixtures
- [ ] T007 [P] Create tests/unit/__init__.py for unit tests
- [ ] T008 [P] Create tests/integration/__init__.py for integration tests
- [ ] T009 [P] Create tests/fixtures/ directory for test data

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T010 [P] Implement Config model in src/ai_comment_sweeper/models/config.py with Pydantic BaseModel
- [ ] T011 [P] Implement LLMConfig model in src/ai_comment_sweeper/models/config.py with provider validation
- [ ] T012 [P] Implement MarkdownConfig model in src/ai_comment_sweeper/models/config.py with sacred files list
- [ ] T013 [P] Implement CacheConfig model in src/ai_comment_sweeper/models/config.py with TTL validation
- [ ] T014 Implement load_config() function in src/ai_comment_sweeper/config.py with JSON loading and validation
- [ ] T015 Implement environment variable resolution in src/ai_comment_sweeper/config.py for ${VAR} patterns
- [ ] T016 [P] Implement SourceFile model in src/ai_comment_sweeper/models/file.py with path and language fields
- [ ] T017 [P] Implement DocumentationFile model in src/ai_comment_sweeper/models/file.py with age calculation
- [ ] T018 [P] Implement SourceProcessingResult model in src/ai_comment_sweeper/models/result.py with action field
- [ ] T019 [P] Implement DocumentationProcessingResult model in src/ai_comment_sweeper/models/result.py with recommendation field
- [ ] T020 [P] Implement CacheEntry model in src/ai_comment_sweeper/models/cache.py with git_commit and content_hash fields
- [ ] T021 [P] Implement CacheManifest model in src/ai_comment_sweeper/models/cache.py with timestamp and TTL
- [ ] T022 [P] Implement ProcessingSummary model in src/ai_comment_sweeper/models/result.py with statistics fields
- [ ] T023 Implement get_repo() function in src/ai_comment_sweeper/safety.py using GitPython Repo
- [ ] T024 Implement verify_git_repository() function in src/ai_comment_sweeper/safety.py with exit on failure
- [ ] T025 Implement get_file_last_commit_time() function in src/ai_comment_sweeper/safety.py using git log
- [ ] T026 Implement get_current_commit_hash() function in src/ai_comment_sweeper/safety.py from repo.head
- [ ] T027 Implement has_uncommitted_changes() function in src/ai_comment_sweeper/safety.py using repo.is_dirty()
- [ ] T028 Implement show_safety_warning() function in src/ai_comment_sweeper/safety.py using Rich Panel
- [ ] T029 Implement LLMClient class in src/ai_comment_sweeper/llm/client.py with LiteLLM integration
- [ ] T030 Implement complete() method in src/ai_comment_sweeper/llm/client.py with retry logic
- [ ] T031 Implement count_tokens() method in src/ai_comment_sweeper/llm/client.py using LiteLLM token_counter
- [ ] T032 Implement estimate_cost() method in src/ai_comment_sweeper/llm/client.py with provider-specific pricing

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Safe Preview of Comment Cleanup (Priority: P1) 🎯 MVP

**Goal**: Enable dry-run mode that shows what would be processed without making changes

**Independent Test**: Run `ai-comment-sweeper --dry-run` on sample repo, verify summary displayed and no files modified

### Implementation for User Story 1

- [ ] T033 [P] [US1] Implement EXTENSION_MAP in src/ai_comment_sweeper/scanner.py with file type mappings (.py, .js, .ts, .md, etc.)
- [ ] T034 [P] [US1] Implement get_file_language() function in src/ai_comment_sweeper/scanner.py using extension mapping with Pygments fallback
- [ ] T035 [P] [US1] Implement is_source_file() function in src/ai_comment_sweeper/scanner.py to identify source code files
- [ ] T036 [P] [US1] Implement is_documentation_file() function in src/ai_comment_sweeper/scanner.py to identify markdown files
- [ ] T037 [US1] Implement scan_repository() function in src/ai_comment_sweeper/scanner.py with .gitignore respect and file categorization
- [ ] T038 [US1] Implement ScanResult model in src/ai_comment_sweeper/models/file.py with source_files and documentation_files lists
- [ ] T039 [US1] Implement estimate_token_count() function in src/ai_comment_sweeper/scanner.py for dry-run cost estimation
- [ ] T040 [US1] Implement estimate_processing_cost() function in src/ai_comment_sweeper/scanner.py based on token counts
- [ ] T041 [US1] Implement display_dry_run_summary() function in src/ai_comment_sweeper/reporter.py using Rich Table
- [ ] T042 [US1] Implement CLI main() function in src/ai_comment_sweeper/cli.py with Click decorators
- [ ] T043 [US1] Add --dry-run flag to CLI in src/ai_comment_sweeper/cli.py with is_flag=True
- [ ] T044 [US1] Implement dry_run_mode() function in src/ai_comment_sweeper/cli.py orchestrating scan and summary

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently with `--dry-run` flag

---

## Phase 4: User Story 2 - Analyze and Cache Comment Cleanup (Priority: P1) 🎯 MVP

**Goal**: Process files with LLM, cache results, display diffs without modifying files

**Independent Test**: Run `ai-comment-sweeper` (default mode), verify cache created in .aisweeper-cache/, diffs displayed, files unchanged

### Implementation for User Story 2

- [ ] T045 [P] [US2] Create CONSERVATIVE_PROMPT template in src/ai_comment_sweeper/llm/prompts.py from contracts/source-processor-prompts.md
- [ ] T046 [P] [US2] Create MODERATE_PROMPT template in src/ai_comment_sweeper/llm/prompts.py from contracts/source-processor-prompts.md
- [ ] T047 [P] [US2] Create AGGRESSIVE_PROMPT template in src/ai_comment_sweeper/llm/prompts.py from contracts/source-processor-prompts.md
- [ ] T048 [P] [US2] Create NUCLEAR_PROMPT template in src/ai_comment_sweeper/llm/prompts.py from contracts/source-processor-prompts.md
- [ ] T049 [P] [US2] Create SOURCE_SYSTEM_PROMPT in src/ai_comment_sweeper/llm/prompts.py for comment curation task
- [ ] T050 [US2] Implement get_source_prompt() function in src/ai_comment_sweeper/llm/prompts.py selecting prompt by aggressiveness level
- [ ] T051 [US2] Implement BaseProcessor abstract class in src/ai_comment_sweeper/processors/base.py with process() method signature
- [ ] T052 [US2] Implement SourceProcessor class in src/ai_comment_sweeper/processors/source.py inheriting from BaseProcessor
- [ ] T053 [US2] Implement process() method in src/ai_comment_sweeper/processors/source.py calling LLM with language-aware prompts
- [ ] T054 [US2] Implement parse_llm_response() method in src/ai_comment_sweeper/processors/source.py handling SKIP and cleaned content
- [ ] T055 [US2] Implement generate_diff() function in src/ai_comment_sweeper/diff.py using difflib.unified_diff
- [ ] T056 [US2] Implement display_diff() function in src/ai_comment_sweeper/diff.py using Rich Syntax with diff theme
- [ ] T057 [US2] Implement parse_diff_stats() function in src/ai_comment_sweeper/diff.py counting +/- lines
- [ ] T058 [US2] Implement hash_content() function in src/ai_comment_sweeper/cache.py using SHA256
- [ ] T059 [US2] Implement hash_file_path() function in src/ai_comment_sweeper/cache.py for cache filename generation
- [ ] T060 [US2] Implement CacheManager class in src/ai_comment_sweeper/cache.py with cache_dir and git_commit fields
- [ ] T061 [US2] Implement write_entry() method in src/ai_comment_sweeper/cache.py with atomic write (tempfile + rename)
- [ ] T062 [US2] Implement read_entry() method in src/ai_comment_sweeper/cache.py with git commit validation
- [ ] T063 [US2] Implement is_cache_valid() method in src/ai_comment_sweeper/cache.py checking commit hash and content hash
- [ ] T064 [US2] Implement create_cache_manifest() function in src/ai_comment_sweeper/cache.py writing manifest.json
- [ ] T065 [US2] Implement default_mode() function in src/ai_comment_sweeper/cli.py orchestrating scan, process, cache, and diff display
- [ ] T066 [US2] Add default mode execution in CLI main() when no flags specified in src/ai_comment_sweeper/cli.py

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently (dry-run + analysis/cache)

---

## Phase 5: User Story 3 - Apply Cached Changes (Priority: P1) 🎯 MVP

**Goal**: Apply cached changes to files after user reviews diffs

**Independent Test**: Run analysis mode to cache, then run `--apply`, verify files modified per diffs

### Implementation for User Story 3

- [ ] T067 [P] [US3] Implement validate_cache_exists() function in src/ai_comment_sweeper/cache.py checking for manifest.json
- [ ] T068 [P] [US3] Implement validate_cache_fresh() function in src/ai_comment_sweeper/cache.py checking TTL and git commit
- [ ] T069 [US3] Implement load_cache_manifest() function in src/ai_comment_sweeper/cache.py reading manifest.json
- [ ] T070 [US3] Implement apply_source_changes() function in src/ai_comment_sweeper/apply.py writing cleaned content to files
- [ ] T071 [US3] Implement apply_cached_changes() function in src/ai_comment_sweeper/apply.py iterating cache entries and applying
- [ ] T072 [US3] Implement verify_file_unchanged() function in src/ai_comment_sweeper/apply.py comparing content hash before apply
- [ ] T073 [US3] Implement display_apply_summary() function in src/ai_comment_sweeper/reporter.py showing files modified stats
- [ ] T074 [US3] Add --apply flag to CLI in src/ai_comment_sweeper/cli.py with is_flag=True
- [ ] T075 [US3] Implement apply_mode() function in src/ai_comment_sweeper/cli.py with cache validation and application
- [ ] T076 [US3] Add mutual exclusion check between --dry-run and --apply in src/ai_comment_sweeper/cli.py

**Checkpoint**: All three P1 user stories should now be independently functional (dry-run, analysis, apply)

---

## Phase 6: User Story 4 - Markdown Documentation Analysis (Priority: P2)

**Goal**: Analyze markdown files for deletion with LLM

**Independent Test**: Run tool with markdown files, verify deletion recommendations generated

### Implementation for User Story 4

- [ ] T077 [P] [US4] Create DOCUMENTATION_SYSTEM_PROMPT in src/ai_comment_sweeper/llm/prompts.py from contracts/documentation-processor-prompts.md
- [ ] T078 [P] [US4] Create DOCUMENTATION_USER_PROMPT template in src/ai_comment_sweeper/llm/prompts.py with file metadata placeholders
- [ ] T079 [US4] Implement get_documentation_prompt() function in src/ai_comment_sweeper/llm/prompts.py formatting with file details
- [ ] T080 [US4] Implement matches_ephemeral_pattern() function in src/ai_comment_sweeper/scanner.py using fnmatch on ephemeral_patterns
- [ ] T081 [US4] Implement is_sacred_file() function in src/ai_comment_sweeper/scanner.py checking against sacred list
- [ ] T082 [US4] Implement calculate_days_since_commit() function in src/ai_comment_sweeper/scanner.py from last_commit_time
- [ ] T083 [US4] Implement DocumentationProcessor class in src/ai_comment_sweeper/processors/documentation.py inheriting BaseProcessor
- [ ] T084 [US4] Implement process() method in src/ai_comment_sweeper/processors/documentation.py calling LLM with file context
- [ ] T085 [US4] Implement parse_json_response() method in src/ai_comment_sweeper/processors/documentation.py parsing recommendation/confidence
- [ ] T086 [US4] Implement should_delete() function in src/ai_comment_sweeper/processors/documentation.py checking HIGH confidence DELETE
- [ ] T087 [US4] Implement display_deletion_recommendations() function in src/ai_comment_sweeper/reporter.py showing files to delete with reasoning
- [ ] T088 [US4] Implement apply_documentation_deletions() function in src/ai_comment_sweeper/apply.py removing HIGH confidence DELETE files
- [ ] T089 [US4] Add --markdown-only flag to CLI in src/ai_comment_sweeper/cli.py filtering to .md files only
- [ ] T090 [US4] Integrate DocumentationProcessor into default_mode() in src/ai_comment_sweeper/cli.py
- [ ] T091 [US4] Integrate doc deletions into apply_mode() in src/ai_comment_sweeper/cli.py

**Checkpoint**: User Story 4 complete - markdown analysis and deletion working

---

## Phase 7: User Story 5 - Configurable Behavior (Priority: P3)

**Goal**: Support custom configuration via .aisweeper.json

**Independent Test**: Create custom config file, verify tool uses custom settings

### Implementation for User Story 5

- [ ] T092 [P] [US5] Implement merge_configs() function in src/ai_comment_sweeper/config.py combining user config with defaults
- [ ] T093 [P] [US5] Implement validate_config_schema() function in src/ai_comment_sweeper/config.py using Pydantic validation
- [ ] T094 [US5] Add config file validation to load_config() in src/ai_comment_sweeper/config.py with clear error messages
- [ ] T095 [US5] Implement display_config_errors() function in src/ai_comment_sweeper/config.py formatting Pydantic ValidationError
- [ ] T096 [US5] Add --config option to CLI in src/ai_comment_sweeper/cli.py for custom config path
- [ ] T097 [US5] Add --aggressiveness option to CLI in src/ai_comment_sweeper/cli.py overriding config file setting
- [ ] T098 [US5] Implement config_mode() function in src/ai_comment_sweeper/cli.py showing current configuration
- [ ] T099 [US5] Add --show-config flag to CLI in src/ai_comment_sweeper/cli.py displaying merged configuration

**Checkpoint**: All user stories complete - full MVP functionality implemented

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T100 [P] Implement display_processing_summary() function in src/ai_comment_sweeper/reporter.py with Rich Table showing all stats
- [ ] T101 [P] Implement format_cost() helper in src/ai_comment_sweeper/reporter.py for USD formatting
- [ ] T102 [P] Implement format_duration() helper in src/ai_comment_sweeper/reporter.py for time formatting
- [ ] T103 [P] Add progress bars to file processing using Rich track() in src/ai_comment_sweeper/cli.py
- [ ] T104 [P] Add error handling for LLM API failures in src/ai_comment_sweeper/llm/client.py with retry exhaustion
- [ ] T105 [P] Add error handling for git operations failures in src/ai_comment_sweeper/safety.py with helpful messages
- [ ] T106 [P] Add --version flag to CLI in src/ai_comment_sweeper/cli.py displaying package version
- [ ] T107 [P] Add --help with comprehensive usage examples in src/ai_comment_sweeper/cli.py
- [ ] T108 Implement graceful degradation for missing git history using filesystem mtime fallback in src/ai_comment_sweeper/safety.py
- [ ] T109 Implement file size limit check in src/ai_comment_sweeper/scanner.py warning for files >50KB
- [ ] T110 Add logging infrastructure in src/ai_comment_sweeper/__init__.py using Python logging module

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-7)**: All depend on Foundational phase completion
  - User Story 1 (P1): Dry-run mode - independent
  - User Story 2 (P1): Analysis/cache - independent
  - User Story 3 (P1): Apply mode - depends on US2 cache infrastructure
  - User Story 4 (P2): Markdown analysis - independent
  - User Story 5 (P3): Configuration - independent
- **Polish (Phase 8)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational - No dependencies on other stories
- **User Story 2 (P1)**: Can start after Foundational - No dependencies on other stories
- **User Story 3 (P1)**: Requires US2 cache infrastructure (CacheManager, write_entry, read_entry)
- **User Story 4 (P2)**: Can start after Foundational - No dependencies on other stories
- **User Story 5 (P3)**: Can start after Foundational - No dependencies on other stories

### Within Each User Story

- Models before services
- Services before CLI integration
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- **Setup phase**: Tasks T002-T009 can run in parallel (different directories)
- **Foundational phase**:
  - Model tasks T010-T022 can run in parallel (different model files)
  - Safety tasks T023-T028 can run in parallel
  - LLM tasks T029-T032 can run in parallel
- **User Story 1**: Tasks T033-T036 can run in parallel (scanner utilities)
- **User Story 2**:
  - Prompt tasks T045-T049 can run in parallel
  - Cache tasks T058-T059 can run in parallel
- **User Story 4**: Prompt tasks T077-T078 can run in parallel, scanner tasks T080-T082 can run in parallel
- **User Story 5**: Tasks T092-T093 can run in parallel
- **Polish phase**: Tasks T100-T107 can run in parallel (different concerns)

---

## Parallel Example: User Story 2

```bash
# Launch prompt template tasks together:
Task T045: "Create CONSERVATIVE_PROMPT in src/ai_comment_sweeper/llm/prompts.py"
Task T046: "Create MODERATE_PROMPT in src/ai_comment_sweeper/llm/prompts.py"
Task T047: "Create AGGRESSIVE_PROMPT in src/ai_comment_sweeper/llm/prompts.py"
Task T048: "Create NUCLEAR_PROMPT in src/ai_comment_sweeper/llm/prompts.py"
Task T049: "Create SOURCE_SYSTEM_PROMPT in src/ai_comment_sweeper/llm/prompts.py"

# Launch cache utility tasks together:
Task T058: "Implement hash_content() in src/ai_comment_sweeper/cache.py"
Task T059: "Implement hash_file_path() in src/ai_comment_sweeper/cache.py"
```

---

## Implementation Strategy

### MVP First (User Stories 1-3 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1 (Dry-run)
4. **STOP and VALIDATE**: Test dry-run mode independently
5. Complete Phase 4: User Story 2 (Analysis/Cache)
6. **STOP and VALIDATE**: Test analysis mode independently
7. Complete Phase 5: User Story 3 (Apply)
8. **STOP and VALIDATE**: Test full workflow (dry-run → analyze → apply)
9. Deploy/demo if ready

**MVP Scope**: Phases 1-5 (110 tasks total: 9 setup + 23 foundational + 12 US1 + 22 US2 + 10 US3)

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (Dry-run capability!)
3. Add User Story 2 → Test independently → Deploy/Demo (Analysis capability!)
4. Add User Story 3 → Test independently → Deploy/Demo (Full MVP!)
5. Add User Story 4 → Test independently → Deploy/Demo (Markdown cleanup!)
6. Add User Story 5 → Test independently → Deploy/Demo (Configuration!)
7. Add Phase 8 (Polish) → Final release

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1 (T033-T044)
   - Developer B: User Story 2 (T045-T066)
   - Developer C: User Story 4 (T077-T091)
3. Developer A picks up User Story 3 after US1 complete (requires US2 cache)
4. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- All file paths use src/ai_comment_sweeper/ prefix for clarity
- Models defined before used by services
- Services integrated into CLI after implementation complete
