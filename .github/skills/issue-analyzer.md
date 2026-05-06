# Skill: Issue Analysis

## Purpose
Parse GitHub issues and extract actionable requirements for the sysinfo agent.

## When to Use
Every time a `/solve` command is triggered on an issue.

## Process

### 1. Extract Metadata
- Issue title
- Issue body
- Issue labels (look for `auto-solve`, `bug`, `feature`, `documentation`, platform labels)
- Issue number & URL
- Author

### 2. Identify Problem Statement
Extract the core problem(s):
- What is broken or missing?
- What platforms are affected? (Linux, macOS, Windows, BSD, iOS, Android, or all?)
- What is the expected behavior?
- What is the actual behavior?

### 3. Identify Requirements
- What code changes are needed?
- What tests should be added?
- What documentation should be updated?
- What feature gates are involved?

### 4. Assess Complexity
- **Level 1** (POC): Docs, tests, simple fixes (<100 lines, single file)
- **Level 2** (Medium): Bug fixes, feature additions (100-500 lines, possibly multi-file)
- **Level 3** (Complex): Major refactors, new platforms (500+ lines, multi-file, architecture)

### 5. Check Auto-Solve Eligibility
- ✅ **Can auto-solve**: Level 1 + has `auto-solve` label
- ❓ **Requires approval**: Level 2+ or missing `auto-solve` label
- ❌ **Cannot solve**: Breaking changes, MSRV violations, unsafe code without context

### 6. Output Summary
```markdown
## Analysis Summary
**Issue**: [#123 Title](url)
**Problem**: [Clear statement]
**Platforms**: [Linux, macOS, Windows]
**Complexity**: Level 1/2/3
**Auto-Solve**: ✅ Yes / ❓ Needs Approval / ❌ Cannot Solve
**Requirements**:
- [ ] Change 1
- [ ] Change 2
- [ ] Test updates
```

## Example
**Input**: Issue #150 "Process memory reporting shows 0 on macOS"

**Output**:
```markdown
## Analysis Summary
**Issue**: #150 Process memory reporting shows 0 on macOS
**Problem**: `Process::memory()` returns 0 instead of actual memory usage on macOS
**Platforms**: macOS
**Complexity**: Level 2 (bug fix, platform-specific code)
**Auto-Solve**: ❓ Requires approval (Level 2)
**Requirements**:
- [ ] Check macOS FFI calls for memory info
- [ ] Fix resident/virtual memory calculation
- [ ] Add test case for macOS memory reporting
```
