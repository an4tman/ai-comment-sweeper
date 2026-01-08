# Source Processor LLM Prompts

**Purpose**: Prompt templates for source file comment cleaning at different aggressiveness levels.

---

## System Prompt (All Levels)

```
You are a code comment curator specializing in removing AI-generated meta-commentary while preserving valuable documentation.

Your task is to clean source code comments by removing:
1. AI meta-commentary about the coding process
2. References to previous code iterations
3. Explanations of changes that were made
4. Stale TODO items from AI suggestions

You must preserve:
1. Documentation of function/class behavior
2. Non-obvious implementation details
3. UX/design rationale
4. Security, performance, or edge-case warnings
5. Public API documentation

If no changes are needed, return exactly: SKIP

Otherwise, return the complete cleaned file content.
```

---

## Conservative Level

**User Prompt Template**:
```
Language: {language}
Aggressiveness: Conservative

Remove ONLY obvious AI meta-commentary from this {language} file:

Examples of what to remove:
- "I'll now add..."
- "Let me create..."
- "Note: I updated this to..."
- "Claude: This function..."
- "AI: Fixed the bug by..."

Keep everything else, even if redundant.

File content:
```
{file_content}
```

Return the cleaned file or SKIP if no changes needed.
```

**Characteristics**:
- Removes only blatant AI self-references
- Preserves all implementation comments
- Preserves all TODOs
- Preserves redundant comments
- Very low false positive risk

---

## Moderate Level (Default)

**User Prompt Template**:
```
Language: {language}
Aggressiveness: Moderate

Remove AI-generated meta-commentary and stale context from this {language} file:

Remove:
- AI self-references ("I'll...", "Let me...", "Note: I updated...")
- References to previous iterations ("This was changed from...", "Previously we had...")
- Change explanations ("Updated to fix X", "Refactored to improve Y")
- Stale TODOs from AI suggestions (keep user TODOs)
- AI reasoning ("Since we need X, I added Y")

Preserve:
- Function/class documentation (docstrings, JSDoc)
- Implementation notes about non-obvious logic
- Security/performance warnings
- Edge case explanations
- UX/design rationale

File content:
```
{file_content}
```

Return the cleaned file or SKIP if no changes needed.
```

**Characteristics**:
- Removes AI meta-commentary and iteration references
- Removes stale TODOs that look AI-generated
- Preserves meaningful implementation notes
- Moderate false positive risk, good for most codebases

---

## Aggressive Level

**User Prompt Template**:
```
Language: {language}
Aggressiveness: Aggressive

Aggressively clean comments from this {language} file:

Remove:
- All AI meta-commentary
- All iteration references
- All change explanations
- All stale TODOs
- Redundant comments that merely restate the code
- Comments explaining standard patterns
- Self-evident implementation notes

Preserve ONLY:
- Non-obvious algorithmic explanations
- Security/performance warnings
- Complex edge cases
- Public API documentation (docstrings, JSDoc for public methods)
- Required compliance comments (license headers, attributions)

If a comment doesn't add information beyond what the code clearly shows, remove it.

File content:
```
{file_content}
```

Return the cleaned file or SKIP if no changes needed.
```

**Characteristics**:
- Removes redundant comments that restate code
- Removes standard implementation explanations
- Preserves only truly valuable comments
- Higher false positive risk, use for production-ready code

---

## Nuclear Level

**User Prompt Template**:
```
Language: {language}
Aggressiveness: Nuclear

Ruthlessly question every comment in this {language} file:

Remove everything EXCEPT:
- Complex algorithm explanations (e.g., optimization techniques, mathematical formulas)
- Critical security warnings (e.g., "XSS risk if...", "Timing attack possible...")
- Performance constraints (e.g., "Must be O(1)", "Cached for 10K+ items")
- Public API documentation for exported/public methods ONLY
- License headers and legal attributions

Every comment must justify its existence. If the code is readable, remove the comment.

This level assumes:
- Clean, self-documenting code
- Modern language features (type hints, descriptive names)
- Professional developers who can read code

File content:
```
{file_content}
```

Return the cleaned file or SKIP if no changes needed.
```

**Characteristics**:
- Removes almost all comments
- Assumes code is self-documenting
- Preserves only critical warnings and complex algorithms
- High false positive risk, use only for mature, well-written code

---

## Language-Specific Notes

### Python
- Preserve docstrings for public functions/classes
- Preserve type hint comments for Python <3.5 compatibility (if present)
- Remove AI-generated TODOs but keep user TODOs

### JavaScript/TypeScript
- Preserve JSDoc for exported functions
- Preserve copyright headers
- Remove console.log explanations added by AI

### Go
- Preserve package-level comments
- Preserve exported function comments (Go convention)
- Remove AI reasoning about error handling

### Rust
- Preserve `//!` and `///` doc comments
- Preserve `// SAFETY:` comments
- Remove AI explanations of borrow checker fixes

---

## Example Transformations

### Conservative

**Before**:
```python
# I'll add a helper function to validate email addresses
# Claude: This uses a simple regex for basic validation
def validate_email(email: str) -> bool:
    # Check if email matches pattern
    return re.match(r'^[a-z]+@[a-z]+\.[a-z]+$', email) is not None
```

**After**:
```python
# This uses a simple regex for basic validation
def validate_email(email: str) -> bool:
    # Check if email matches pattern
    return re.match(r'^[a-z]+@[a-z]+\.[a-z]+$', email) is not None
```

**Removed**: AI self-references only

---

### Moderate

**Before**:
```python
# I'll add a helper function to validate email addresses
# Claude: This uses a simple regex for basic validation
def validate_email(email: str) -> bool:
    # Check if email matches pattern
    return re.match(r'^[a-z]+@[a-z]+\.[a-z]+$', email) is not None
```

**After**:
```python
def validate_email(email: str) -> bool:
    # Check if email matches pattern
    return re.match(r'^[a-z]+@[a-z]+\.[a-z]+$', email) is not None
```

**Removed**: AI self-references and redundant docstring

---

### Aggressive

**Before**:
```python
# I'll add a helper function to validate email addresses
# Claude: This uses a simple regex for basic validation
def validate_email(email: str) -> bool:
    # Check if email matches pattern
    return re.match(r'^[a-z]+@[a-z]+\.[a-z]+$', email) is not None
```

**After**:
```python
def validate_email(email: str) -> bool:
    return re.match(r'^[a-z]+@[a-z]+\.[a-z]+$', email) is not None
```

**Removed**: All comments (code is self-documenting with function name)

---

### Nuclear

**Before**:
```python
# I'll add a helper function to validate email addresses
# Claude: This uses a simple regex for basic validation
def validate_email(email: str) -> bool:
    # Check if email matches pattern
    return re.match(r'^[a-z]+@[a-z]+\.[a-z]+$', email) is not None
```

**After**:
```python
def validate_email(email: str) -> bool:
    return re.match(r'^[a-z]+@[a-z]+\.[a-z]+$', email) is not None
```

**Removed**: All comments (not critical security/performance/algorithm)

---

## Response Format

**Option 1: Changes Made**
Return the complete cleaned file content as plain text.

**Option 2: No Changes Needed**
Return exactly: `SKIP`

Do not return explanations, diffs, or JSON. Return only the cleaned file or "SKIP".
