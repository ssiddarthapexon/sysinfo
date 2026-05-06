# Copilot Instructions for sysinfo

## Overview
This file defines the automated workflow for issue resolution, code quality, and pull request management in the sysinfo repository.

---

## Branch Naming Convention

All branches **must** follow this pattern:
```
{type}/{issue-number}-{short-description}
```

### Allowed Types
- `fix/` — Bug fixes (maps to `[FIX]` PR label)
- `feat/` — New features (maps to `[FEAT]` PR label)
- `docs/` — Documentation improvements (maps to `[DOCS]` PR label)
- `test/` — Test additions/improvements (maps to `[TEST]` PR label)
- `refactor/` — Code refactoring (maps to `[REFACTOR]` PR label)

### Examples
```
fix/42-memory-calculation-overflow
feat/15-support-arm64-systems
docs/8-update-readme
test/23-add-cpu-stress-tests
refactor/31-simplify-network-module
```

---

## Pull Request Format

### Automatic PR Creation
When you push a branch matching the naming convention above:

1. **`.github/workflows/auto-pr.yml` triggers automatically**
2. **PR is created with**:
   - Title: `[TYPE] description (#issue-number)`
   - Body: Auto-generated with issue context and validation checklist
   - Labels: `automated` + type-specific labels
   - Base branch: `main`
   - Draft: `false` (ready for review)

### PR Title Format
```
[FIX] memory calculation overflow (#42)
[FEAT] support arm64 systems (#15)
[DOCS] update readme (#8)
```

### PR Body Must Include
```markdown
## Description
Brief summary of changes

## Related Issue
Closes #{issue-number}

## Type of Change
- [x] Bug fix
- [ ] New feature
- [ ] Documentation
- [ ] Breaking change

## Testing
How have you tested this? (e.g., `cargo test --all-features`)

## Checklist
- [x] Code follows style guidelines
- [x] Tests added/updated
- [x] Documentation updated
- [x] No new warnings
```

---

## Code Quality Standards

### Rust Code Style
- **Formatter**: `rustfmt` (enforced in CI)
- **Linter**: `clippy` with all lints enabled
- **MSRV**: 1.95.0 (minimum supported Rust version)
- **Features**: All features must compile and pass tests

### Before Committing
```bash
# Format code
cargo fmt --all

# Lint and check
cargo clippy --all-targets --all-features -- -D warnings

# Run tests
cargo test --all-features

# Run benchmarks (if modified)
cargo bench --bench basic
```

### Commit Message Format
```
{type}: issue #{number} - {short description}

Detailed explanation (if needed)
```

**Examples:**
```
fix: issue #42 - resolve memory calculation overflow

The calculation was using signed integers, causing overflow
on systems with >2GB memory. Changed to u64.

feat: issue #15 - add arm64 support

Added platform detection and arm64-specific code paths
for the CPU and disk modules.
```

---

## Automated Workflows

### 1. **Auto PR Creation** (`.github/workflows/auto-pr.yml`)
- **Trigger**: Push to `fix/*`, `feat/*`, `docs/*`, `test/*`, `refactor/*` branches
- **Actions**:
  - Extracts issue number from branch name
  - Checks if PR already exists
  - Creates PR with auto-generated title and body
  - Adds appropriate labels

### 2. **Solver Validation** (`.github/workflows/solver-validation.yml`)
- **Trigger**: PR created
- **Checks**:
  - ✓ Code formatting (rustfmt)
  - ✓ Linting (clippy)
  - ✓ MSRV compatibility
  - ✓ All features compile
  - ✓ Full test suite passes
  - ✓ Benchmark integrity

### 3. **Issue Fixer Agent** (`.github/agents/issue-fixer.agent.md`)
- **Use when**: User says "fix issue #X" or provides issue URL
- **Does**:
  1. Parse issue reference
  2. Analyze issue description
  3. Identify affected files
  4. Apply code fix
  5. Create branch and commit
  6. Push and create PR

### 4. **Code Explorer Skill** (`.github/skills/code-explorer.md`)
- **Use for**: Understanding codebase structure, finding related files

### 5. **Test Validator Skill** (`.github/skills/test-validator.md`)
- **Use before**: Committing, to ensure all tests pass locally

---

## Workflow in Action

### Scenario: Fixing a Bug

1. **User says**: "Fix issue #42"
   - Agent: `#issue-fixer` activates
   
2. **Agent**:
   - Fetches issue #42 from GitHub
   - Reads relevant code files
   - Applies fix
   - Creates branch: `fix/42-memory-calculation-overflow`
   - Commits: `Fix: issue #42 - resolve memory overflow`
   
3. **User pushes**:
   ```bash
   git push -u origin fix/42-memory-calculation-overflow
   ```
   
4. **Automatic**:
   - `.github/workflows/auto-pr.yml` creates PR: `[FIX] memory calculation overflow (#42)`
   - `.github/workflows/solver-validation.yml` runs all checks
   - PR appears in the UI, ready for review
   
5. **Merge**:
   - Once approved and CI passes, merge to `main`
   - GitHub auto-closes issue #42 (due to "Closes #42" in PR body)

---

## Best Practices

### When Creating Fixes
1. ✅ Always extract issue number in branch name
2. ✅ Reference the issue number in commit messages (`Fix: issue #X`)
3. ✅ Run `cargo test --all-features` before pushing
4. ✅ Keep commits focused on a single issue
5. ✅ If fix is complex, add comments explaining the "why"

### Code Review Guidelines
1. ✅ Check that branch name matches issue
2. ✅ Verify PR title follows format
3. ✅ Confirm all CI checks pass (green checkmarks)
4. ✅ Review the actual code changes
5. ✅ Test locally if significant logic changes

### When Validation Fails
1. Fix the issues locally:
   ```bash
   cargo fmt --all
   cargo clippy --fix
   cargo test --all-features
   ```
2. Commit and push:
   ```bash
   git commit -am "Fix: linting and formatting"
   git push
   ```
3. PR automatically updates and re-runs CI

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `.github/copilot-instructions.md` | This file — foundational behavior rules |
| `.github/workflows/auto-pr.yml` | Auto-creates PR when branch is pushed |
| `.github/workflows/solver-validation.yml` | Runs CI checks on PR |
| `.github/agents/issue-fixer.agent.md` | Agent for fixing issues |
| `.github/agents/pr-automation.md` | Documents the auto-PR system |
| `.github/agents/sysinfo-expert.md` | Specialized sysinfo domain knowledge |
| `.github/skills/code-explorer.md` | Skill for codebase navigation |
| `.github/skills/test-validator.md` | Skill for running tests |

---

## Troubleshooting

### PR Not Creating Automatically
- **Check**: Branch name follows pattern: `{type}/{number}-{desc}`
- **Check**: Issue number is included and valid
- **Check**: `.github/workflows/auto-pr.yml` exists and has no syntax errors
- **Fix**: Push again after fixing branch name

### CI Validation Failing
- **Run locally**: `cargo fmt`, `cargo clippy --fix`, `cargo test --all-features`
- **Commit fixes**: `git commit -am "Fix: ci issues"`
- **Push**: `git push` (PR updates automatically)

### Issue Not Auto-Closing
- **Check**: PR body includes `Closes #{issue-number}`
- **Check**: PR is against `main` branch
- **Manual close**: Merge PR to trigger GitHub's auto-close

---

## Questions?
- Check `.github/README.md` for workflow documentation
- See individual agent/skill files for detailed instructions
- Review `CHANGELOG.md` for version history
