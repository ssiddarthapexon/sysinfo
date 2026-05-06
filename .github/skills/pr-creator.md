# Skill: PR Creator

## Purpose
Generate a high-quality pull request with proper commit message, description, and metadata.

## When to Use
After validation passes, ready to create the PR on GitHub.

## PR Structure

### 1. Branch Naming
**Format**: `{type}/{issue-number}-{short-description}`

**Types**:
- `fix/` — Bug fixes
- `feat/` — Feature additions
- `docs/` — Documentation
- `test/` — Tests
- `refactor/` — Code refactoring

**Examples**:
- `fix/150-process-memory-macos`
- `feat/152-add-gpu-support`
- `docs/148-update-examples`
- `test/151-cpu-coverage`

### 2. Commit Message Format

**Single commit for simple fixes:**
```
[FIX] Process memory reporting returns 0 on macOS (#150)

- Fixed resident memory calculation using libproc
- Added test to validate memory > 0 for current process
- Verified on macOS 12.x and 13.x

Fixes #150
```

**Multiple commits for complex features:**
```
Commit 1: [FEAT] Add base GPU detection API (#152)
Commit 2: [FEAT] Implement NVIDIA GPU support
Commit 3: [TEST] Add GPU detection tests

In final PR description, reference all commits.
```

### Commit Message Components
- **Header**: `[TYPE] Short description` (50 chars max)
- **Body**: 
  - Blank line after header
  - Explain *what* and *why*, not *how*
  - Wrap at 72 characters
  - Reference issue: `Fixes #123` or `Resolves #123`
  - Multiple issues: `Fixes #123, fixes #124`
- **Co-authored**: If multiple people
  ```
  Co-authored-by: Name <email@example.com>
  ```

### Example Commit
```
[FIX] Handle zero-length processes in memory reporting

Some processes report zero memory on certain platforms. This was
causing confusion. Now we explicitly check for zero and log a
debug message instead of silently returning 0.

Fixes #150
Co-authored-by: Reviewer <reviewer@example.com>
```

## PR Description Template

```markdown
## Description
Brief summary of changes (2-3 sentences).

## Related Issue
Fixes #150

## Type of Change
- [x] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to change)
- [ ] Documentation update

## Changes Made
- Change 1
- Change 2
- Change 3

## Testing
Describe how you tested the changes:
- [x] Tested on Linux
- [x] Tested on macOS
- [x] Tested on Windows
- [ ] Tested on iOS
- [ ] Tested on Android

## Validation Checklist
- [x] Code follows sysinfo style guidelines
- [x] `cargo fmt -- --check` passes
- [x] `cargo clippy -- -D warnings` passes
- [x] `cargo test` passes on all platforms
- [x] No new compiler warnings
- [x] New tests added
- [x] Documentation updated
- [x] Backwards compatible (no breaking changes)

## Additional Notes
Any additional context, screenshots, or links?
```

## Key Elements

### Title Format
```
[TYPE] Short description (reference issue)

Examples:
- [FIX] Process memory returns 0 on macOS (#150)
- [FEAT] Add GPU information API (#152)
- [DOCS] Update README examples (#148)
- [TEST] Add missing CPU tests (#151)
```

### Body Requirements
- ✅ Explain the problem (for fixes)
- ✅ Explain the solution
- ✅ Link related issues (`Fixes #123`)
- ✅ Describe testing performed
- ✅ Mention platforms tested
- ✅ Note any limitations or future work
- ✅ Reference existing code patterns used

### Platform Testing Statement
```markdown
## Platform Testing
- [x] Compiled on MSRV (1.95.0)
- [x] Linux (x86_64): Tests pass
- [x] macOS (Intel + Apple Silicon): Tests pass
- [x] Windows (MSVC): Tests pass
- [ ] iOS (requires Apple hardware)
- [ ] Android (requires toolchain setup)
```

## Draft vs. Regular PR

### When to Create Draft PR
- Issue is Level 2 or 3 (not simple)
- Missing `auto-solve` label
- Requires architecture review
- Multiple platforms need human testing

**Prefix title with**: `[DRAFT]` or mark as draft in GitHub

### When to Create Regular PR
- Issue is Level 1 with `auto-solve` label
- Simple, self-explanatory fix
- Full test validation passed
- No architectural concerns

## Labels to Add

**Auto-add to PR**:
- `bug` — if fixing a bug
- `feature` — if adding feature
- `documentation` — if docs
- `platform/linux`, `platform/macos`, `platform/windows` — based on affected platforms
- `good-first-issue` — if suitable for newcomers (rarely applies to agent-generated)

## CI Integration Note

The PR will trigger automatic CI on creation:
1. **rustfmt** job runs
2. **clippy** job runs
3. **check** job runs (all targets)
4. **tests** job runs (all platforms)
5. **c_interface** & **unknown-targets** jobs run

**PR can merge only after all checks pass** ✅

## Review Requirements

While agent creates PR automatically, human review is required:
- [x] Code quality acceptable?
- [x] Tests adequate?
- [x] Commit message clear?
- [x] Issue properly resolved?
- [x] No regressions introduced?

## Example PR (Complete)

```
Title: [FIX] Process memory reporting returns 0 on macOS (#150)

## Description
Fixed issue where `Process::memory()` returns 0 on macOS even for running processes with allocated memory.

## Related Issue
Fixes #150

## Type of Change
- [x] Bug fix (non-breaking change that fixes an issue)

## Changes Made
- Fixed resident memory calculation in `src/unix/apple/process.rs`
- Now correctly uses `proc_pidinfo` with `PROC_PIDREGIONINFO` selector
- Added regression test in `tests/process.rs`

## Testing
- [x] macOS 12.x (Intel): Verified process memory > 0
- [x] macOS 13.x (Apple Silicon): Verified process memory > 0
- [x] Linux: Verified no regression
- [x] Windows: Verified no regression
- [x] All feature combinations build successfully

## Validation Checklist
- [x] `cargo fmt` passes
- [x] `cargo clippy -- -D warnings` passes
- [x] `cargo +1.95.0 check` passes
- [x] `cargo test` passes
- [x] Tests added

## Notes
Tested with Activity Monitor open to confirm memory values align.

---

Commit:
```
[FIX] Process memory returns 0 on macOS

Fixed resident memory calculation using correct libproc selector.
Previously used wrong info struct, now uses proc_pidinfo with
PROC_PIDREGIONINFO which correctly reports memory usage.

Fixes #150
```
