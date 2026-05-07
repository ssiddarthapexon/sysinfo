# Available Agents

This document describes all available agents for the Copilot workflow. Agents are invoked using the `/agent-name` syntax in GitHub Copilot Chat.

---

## Core Specification & Planning Workflow

### `/specify`
**Create or update feature specification from natural language description**

Transforms a feature description into a structured specification document.

- **Input**: Feature description or requirements
- **Output**: `docs/{FEATURE_NUM}-{feature-name}/spec.md`
- **Next Steps**: `/clarify` or `/plan`
- **Time to run**: 5-10 minutes

---

### `/clarify`
**Identify and resolve ambiguous areas in specification**

Conducts up to 5 targeted clarification questions to reduce specification ambiguity before planning.

- **Input**: Existing `spec.md` file
- **Output**: Updated `spec.md` with clarifications recorded
- **Prerequisites**: Must run after `/specify`
- **Next Steps**: `/plan`
- **Time to run**: 5-15 minutes

---

### `/plan`
**Create implementation plan from specification**

Develops a detailed technical implementation plan with architecture, data model, and phased approach.

- **Input**: `spec.md`, optional constitution
- **Output**: `docs/{FEATURE_NUM}-{feature-name}/plan.md`, research.md, data-model.md, contracts/api.md, quickstart.md
- **Prerequisites**: Must run after `/specify`
- **Next Steps**: `/tasks`
- **Time to run**: 10-20 minutes

---

### `/tasks`
**Generate actionable, dependency-ordered tasks**

Breaks down the implementation plan into granular, executable tasks with phase grouping and parallelization markers.

- **Input**: `spec.md`, `plan.md`
- **Output**: `docs/{FEATURE_NUM}-{feature-name}/tasks.md`
- **Prerequisites**: Must run after `/plan`
- **Next Steps**: `/analyze` or `/implement`
- **Time to run**: 5-10 minutes

---

## Quality Assurance & Analysis

### `/analyze`
**Cross-artifact consistency analysis**

Performs non-destructive analysis of spec.md, plan.md, and tasks.md for inconsistencies, duplications, ambiguities, and coverage gaps.

- **Input**: `spec.md`, `plan.md`, `tasks.md`
- **Output**: Markdown analysis report (no file modifications)
- **Prerequisites**: Must run after `/tasks`
- **Report includes**: 
  - Duplication detection
  - Ambiguity detection
  - Coverage gaps
  - Constitution alignment
  - Severity-based findings
- **Time to run**: 5-10 minutes

---

### `/checklist`
**Generate requirement checklists**

Creates domain-specific checklists for validating specification and plan quality.

- **Input**: Feature description or existing specifications
- **Output**: `docs/{FEATURE_NUM}-{feature-name}/checklists/{checklist-name}.md`
- **Use cases**: 
  - Requirements quality validation
  - Architecture design review
  - Security checklist
  - Performance checklist
- **Time to run**: 5-10 minutes

---

## Implementation

### `/implement`
**Execute implementation plan**

Processes tasks from tasks.md and executes the implementation, creating code and artifacts according to the specification.

- **Input**: `spec.md`, `plan.md`, `tasks.md`
- **Output**: Source code, documentation, test files
- **Prerequisites**: Requires complete task breakdown in `tasks.md`
- **Note**: Works iteratively, tracking progress through task phases
- **Time to run**: Variable (depends on complexity)

---

## Project Constitution & Configuration

### `/constitution`
**Create or update project constitution**

Defines project-wide principles, values, and quality gates that constrain all feature development.

- **Input**: Project principles and constraints
- **Output**: `.state/memory/constitution.md`
- **Usage**: Referenced by `/analyze` to validate architecture and design decisions
- **Scope**: Project-level, applies to all features
- **Time to run**: 5-10 minutes

---

## Git Workflow Agents

### `/git.initialize`
**Initialize Git repository**

Sets up a new Git repository with initial commit and configuration.

- **Input**: Project name and description
- **Output**: Initialized `.git/` directory
- **Prerequisites**: Run once per new project
- **Next Steps**: `/git.feature` for feature branches
- **Time to run**: 1-2 minutes

---

### `/git.feature`
**Create and switch to feature branch**

Creates a new feature branch following configured naming conventions (sequential or timestamp-based).

- **Input**: Feature short name and description
- **Output**: New git branch, checked out locally
- **Usage**: Run after `/specify` to create isolated branch for feature development
- **Config**: `.state/extensions/git/git-config.yml`
- **Time to run**: 1 minute

---

### `/git.validate`
**Validate branch naming conventions**

Checks if current branch follows project naming conventions.

- **Input**: Current git branch
- **Output**: Validation report with suggestions
- **Use when**: Verifying branch naming before PR
- **Time to run**: < 1 minute

---

### `/git.commit`
**Auto-commit changes**

Stages and commits all changes with intelligent commit messages.

- **Input**: Changed files (auto-detected)
- **Output**: New git commit
- **Usage**: Can run as hook after other agents or manually
- **Config**: `.state/extensions/git/git-config.yml`
- **Time to run**: 1-2 minutes

---

### `/git.remote`
**Detect Git remote URL**

Identifies and validates the Git remote repository URL.

- **Input**: Current git repository
- **Output**: Remote URL and validation result
- **Use when**: Confirming push destination before PR
- **Time to run**: < 1 minute

---

## Recommended Workflow Sequences

### **Full Feature Development**
```
/git.feature → /specify → /clarify → /plan → /tasks → /analyze → /implement → /git.commit
```

### **Quick Feature** (skip clarification)
```
/specify → /plan → /tasks → /analyze → /implement → /git.commit
```

### **Specification-Only** (planning phase)
```
/specify → /clarify → /plan → /checklist
```

### **Analysis & Refinement**
```
/analyze (review artifacts) → /checklist (validation) → /plan (adjust if needed)
```

---

## Configuration Files

These agents reference optional configuration files:

- **`.state/extensions.yml`** — Hooks for pre/post agent execution
- **`.state/memory/constitution.md`** — Project principles for validation
- **`.state/templates/*.md`** — Templates for spec, plan, tasks, checklist
- **`.state/extensions/git/git-config.yml`** — Git branch naming and commit conventions
- **`.state/scripts/powershell/*.ps1`** — Prerequisite checking and setup scripts

---

## Troubleshooting

**Agent not found?**
- Verify agent file exists in `.github/agents/{agent-name}.agent.md`
- Check VS Code Copilot extension is up to date

**Agent runs but hangs?**
- Check if prerequisite scripts (`.state/scripts/powershell/`) exist
- Verify required files are present (spec.md, plan.md, etc.)

**Need custom commands?**
- Create custom agent files in `.github/agents/`
- Add prompts to `.github/prompts/`
- Document in this file

---

## Agent Naming Convention

Agent files follow this pattern:
- **Core agents**: `{agent-name}.agent.md` (e.g., `specify.agent.md`)
- **Namespaced agents**: `{namespace}.{agent-name}.agent.md` (e.g., `git.feature.agent.md`)

Agents are invoked using just the agent name: `/specify`, `/git.feature`, etc.
