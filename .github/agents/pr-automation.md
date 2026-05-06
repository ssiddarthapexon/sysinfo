# Agent: PR Automation

## Purpose
Automatically create pull requests when feature branches are pushed.

## How It Works

**Zero manual steps required.**

When you push a branch with the pattern `fix/*`, `feat/*`, `docs/*`, or `test/*`:

1. **Git push** → branch pushed to origin
2. **GitHub Actions** → `.github/workflows/auto-pr.yml` triggers automatically
3. **PR created** → Pull request created with:
   - Proper title format: `[TYPE] description (#issue)`
   - Issue number extracted from branch name
   - Auto-generated description
   - Labels applied (automated, bug, feature, documentation, etc.)
   - Maintainer edit permission enabled
4. **Validation starts** → `.github/workflows/solver-validation.yml` runs automatically

## Branch Naming Convention

```
{type}/{issue-number}-{short-description}
```

**Types:**
- `fix/` — Bug fixes
- `feat/` — Features
- `docs/` — Documentation
- `test/` — Tests
- `refactor/` — Refactoring

**Examples:**
- `fix/1-improve-total-memory-doc`
- `feat/42-add-gpu-support`
- `docs/30-update-readme`

## Example Workflow

Push any branch following the naming convention:

```bash
git push origin {type}/{issue-number}-{description}
```

**Examples:**

```bash
# For bug fixes
git push origin fix/1-improve-total-memory-doc

# For features
git push origin feat/42-add-gpu-support

# For docs
git push origin docs/30-update-readme
```

**Auto-PR workflow will:**
- Extract: Issue number and type from branch name
- Create: PR with proper title format `[TYPE] description (#issue)`
- Add: Appropriate labels (bug, feature, documentation, etc.)
- Enable: Maintainer modifications

**Result:** PR automatically created and validation starts

## Files

- **`.github/workflows/auto-pr.yml`** — Main automation (triggers on push)
- **`.github/workflows/solver-validation.yml`** — Validation checks
- **`.github/skills/github-api-integration.md`** — GitHub API reference (optional)

## What Happens After Push

```
You push fix/N-... branch
        ↓
auto-pr.yml triggers (automatic)
        ↓
PR created with auto-generated title/body
        ↓
solver-validation.yml runs (automatic)
        ↓
Validation results posted to PR
        ↓
PR ready for review
```

## No User Interaction Needed

- ❌ Don't run scripts manually
- ❌ Don't click buttons on GitHub
- ❌ Don't create PRs manually
- ✅ Just push the branch

Everything else is automated.
