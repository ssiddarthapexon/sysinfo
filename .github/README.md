# Sysinfo Harness - AI-Powered Issue Solver

## Overview

The **Sysinfo Harness** is an automated system that solves GitHub issues and creates pull requests using AI-driven code generation, powered by a curated knowledge base and validation pipeline.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│           GitHub Issue with /solve comment              │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  .github/workflows/issue-solver.yml                      │
│  - Listens for /solve trigger                            │
│  - Extracts issue metadata                               │
│  - Invokes analysis pipeline                             │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  .github/agents/sysinfo-expert.md                        │
│  "The Brain" - AI agent personality & constraints        │
└──────────────────────┬──────────────────────────────────┘
                       │
           ┌───────────┴───────────┐
           │                       │
           ▼                       ▼
┌──────────────────────┐ ┌──────────────────────┐
│   .github/skills/    │ │   .github/skills/    │
│ - issue-analyzer.md  │ │ - code-explorer.md   │
│ - fix-generator.md   │ │ - test-validator.md  │
│ - pr-creator.md      │ │                      │
└──────────────────────┘ └──────────────────────┘
           │                       │
           └───────────┬───────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  Auto-generated PR with:                                 │
│  - Branch created                                        │
│  - Code changes committed                                │
│  - Tests added/updated                                   │
│  - Commit message formatted                              │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  .github/workflows/solver-validation.yml                │
│  "The Validator" - Automatic quality checks             │
│  - rustfmt compliance                                    │
│  - clippy warnings                                       │
│  - MSRV compatibility                                    │
│  - Feature combinations                                  │
│  - Unit tests                                            │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  ✅ Validation Summary Posted to PR                      │
│  - Ready for review (if all checks pass)                │
│  - Fixes needed (if any checks fail)                    │
└─────────────────────────────────────────────────────────┘
```

## Components

### 1. Agent (`.github/agents/`)

**File**: `sysinfo-expert.md`

The AI agent's personality and constraint system:
- **Role**: Expert Rust systems engineer
- **Knowledge**: sysinfo's multi-platform architecture
- **Constraints**: MSRV 1.95, no breaking changes, API stability
- **Decision Rules**: Complexity levels, refusal conditions, issue triage

### 2. Skills (`.github/skills/`)

Reusable instructions that teach the agent how to perform specific tasks:

- **`issue-analyzer.md`**: Parse GitHub issues, extract requirements, assess complexity
- **`code-explorer.md`**: Navigate sysinfo's codebase, locate implementations, understand patterns
- **`fix-generator.md`**: Write code following sysinfo conventions, handle MSRV, feature gates
- **`test-validator.md`**: Validate against CI pipeline requirements, feature combinations, test matrix
- **`pr-creator.md`**: Format commits, write PR descriptions, add labels, reference issues

### 3. Workflows (`.github/workflows/`)

GitHub Actions that orchestrate the system:

- **`issue-solver.yml`**: Main workflow triggered by `/solve` comment
  - Parses issue
  - Creates branch
  - Invokes agent (framework ready for LLM integration)
  - Creates draft PR
  
- **`solver-validation.yml`**: Validates generated code
  - Checks code formatting (rustfmt)
  - Runs linter (clippy)
  - Validates MSRV (1.95)
  - Tests all feature combinations
  - Runs full test suite
  - Posts validation summary to PR

## Usage

### For Users: Triggering the Solver

1. **Open an issue** on sysinfo repository
2. **Add comment** with `/solve` command:
   ```
   @sysinfo-solver /solve
   ```
3. **If eligible** (Level 1 + `auto-solve` label):
   - Agent automatically analyzes issue
   - Generates fix
   - Creates PR with changes
4. **If not eligible** (Level 2+ or no label):
   - Agent creates draft PR
   - Awaits explicit approval label
   - Proceeds only when approved

### Issue Labels

- `auto-solve` — Issue is safe for automated solving (Level 1)
- `bug`, `feature`, `documentation` — Issue type
- `platform/linux`, `platform/macos`, `platform/windows` — Affected platforms

### Issue Complexity Levels

**Level 1 (POC)**: Simple fixes (auto-solve enabled if labeled)
- Documentation updates
- Test additions
- Small bug fixes (<100 lines, single file)
- Dependency updates

**Level 2 (Medium)**: Requires approval label
- Multi-file bug fixes
- Feature additions
- Error handling improvements

**Level 3 (Complex)**: Requires explicit approval
- Major refactors
- Architecture changes
- Multi-platform features

## Validation Pipeline

When a PR is created by the solver, automatic validation runs:

| Check | Command | Status |
|-------|---------|--------|
| Format | `cargo fmt -- --check` | ✅ or ❌ |
| Lint | `cargo clippy -- -D warnings` | ✅ or ❌ |
| MSRV | `cargo +1.95.0 check` | ✅ or ❌ |
| Features | Multiple `cargo check --features` | ✅ or ❌ |
| Tests | `cargo test` on all platforms | ✅ or ❌ |

**PR Requirements**:
- ✅ All validation checks must pass
- ✅ Human code review required before merge
- ✅ Commit message must reference issue (#123)
- ✅ Tests must be added for fixes/features

## Key Constraints

### MSRV (Minimum Supported Rust Version)
- **Version**: 1.95.0
- **Enforced by**: CI validation job
- **Violations**: PR fails validation, agent refuses to use newer features

### No Breaking Changes
- All public API changes must be additive
- Enum variants can only be added, not removed
- Trait methods can only be added via default implementations
- Agent refuses breaking changes, proposes alternatives

### Multi-Platform Compatibility
- Code must work on: Linux, macOS, Windows, BSD, iOS, Android
- Platform-specific code must use `#[cfg(target_os = "...")]`
- Fallback implementations in `src/unknown/`

### Feature Gate Awareness
- Respect Cargo.toml feature definitions
- Ensure code compiles with/without features
- Don't enable platform-specific features on wrong platforms

## Integration Points

### For LLM/AI Integration

The workflows are designed to integrate with AI:

1. **Input**: GitHub issue comment with `/solve`
2. **Agent Logic**: Use agent personality + skills (in markdown) to guide AI
3. **Output**: Generated code changes, commit message, PR description
4. **Validation**: Automatic CI checks validate output quality
5. **Feedback**: Comments on PR with validation results

**Example Integration** (pseudo-code):
```python
# Workflow calls LLM API
issue = github.get_issue(number)
agent = load_markdown("/.github/agents/sysinfo-expert.md")
skills = load_markdown_files("/.github/skills/*.md")

prompt = f"""
{agent}

Using the skills below, solve this issue:
{issue.body}

{skills}
"""

response = llm.generate(prompt)  # Call Claude, GPT, Copilot, etc.
code_changes = extract_changes(response)
pr = github.create_pr(branch, code_changes, ...)
```

## Validation Rules

The solver **will refuse** to create a PR if:
- ❌ Issue requires breaking API changes
- ❌ MSRV > 1.95 features are needed
- ❌ Unsafe code without clear justification
- ❌ Level 2+ complexity without approval label
- ❌ Code fails validation checks (formatting, lint, tests)

The solver **will create a draft PR** if:
- ⚠️ Level 2 or higher complexity
- ⚠️ Missing `auto-solve` label
- ⚠️ Needs architecture review

## Workflows Explained

### issue-solver.yml

**Trigger**: GitHub issue comment containing `/solve`

**Steps**:
1. Parse issue metadata (title, body, labels, author)
2. Check if eligible for auto-solve
3. Add 🚀 reaction to comment
4. Post status update to issue
5. Create Git branch for fix
6. Install Rust toolchain
7. Generate fix (framework ready for LLM)
8. Validate locally
9. Push branch
10. Create draft PR

**Outputs**:
- Git branch with fix
- Draft PR created

### solver-validation.yml

**Trigger**: PR created (runs automatically)

**Jobs**:
1. **validate_format**: rustfmt check
2. **validate_lint**: clippy on all platforms
3. **validate_msrv**: Rust 1.95.0 compatibility
4. **validate_features**: Feature combination tests
5. **validate_tests**: Full test suite on all platforms
6. **validation_summary**: Post results to PR

**Output**:
- Validation table with results
- Ready for review (✅) or Fixes needed (⚠️)
- PR fails if any check fails (blocks merge)

## File Structure

```
.github/
├── agents/
│   ├── sysinfo-expert.md              # AI agent personality & knowledge
│   └── sysinfo-expert-prompts/        # (Optional) Sub-prompts for specific tasks
├── skills/
│   ├── issue-analyzer.md              # Parse and analyze issues
│   ├── code-explorer.md               # Navigate codebase
│   ├── fix-generator.md               # Generate safe fixes
│   ├── test-validator.md              # Validate against CI
│   └── pr-creator.md                  # Create quality PRs
└── workflows/
    ├── CI.yml                         # Existing CI pipeline (unchanged)
    ├── issue-solver.yml               # Main solver workflow
    └── solver-validation.yml          # Validation pipeline
```

## Success Criteria

✅ **Framework deployed** when:
- Agent file defines sysinfo expertise
- All skills are documented
- Workflows trigger correctly
- Validation pipeline runs automatically

✅ **Agent integration complete** when:
- LLM API integrated (Claude, GPT-4, Copilot, etc.)
- Agent generates working code
- PR creation is fully automated
- Test validation passes

✅ **POC successful** when:
- 3+ Level 1 issues automatically solved
- All validation checks pass
- Generated code is high quality
- No manual fixes needed

## Next Steps

1. **Test framework**: Create sample issues labeled `auto-solve`
2. **Integrate LLM**: Connect to Claude/GPT API
3. **Iterate**: Refine agent prompts based on results
4. **Scale**: Expand to Level 2/3 issues
5. **Monitor**: Track success rate and quality metrics

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `/solve` not triggering | Check issue has `auto-solve` label |
| Validation failing | Review error comments on PR |
| MSRV check fails | Use only Rust 1.95 stable features |
| Tests timeout on macOS | Ensure code respects thread limits |
| Feature combinations fail | Add missing `#[cfg(feature = "...")]` |

---

**Created by**: sysinfo solver framework  
**Version**: 0.1.0 (POC)  
**Last updated**: 2024
