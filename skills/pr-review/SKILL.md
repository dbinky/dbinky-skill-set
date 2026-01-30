---
name: pr-review
description: Use when reviewing a pull request with multi-persona feedback. Spawns sequential sub-agents (Architect, 10x Engineer, Security Expert, Engineering Manager) who post comments to the PR, then interactively lets you choose which fixes to implement.
---

# Multi-Persona PR Review

Orchestrate a comprehensive PR review by spawning specialized sub-agents sequentially. Each reviewer posts comments directly to the PR and can see previous reviewers' feedback.

## Usage

```
/pr-review              # Auto-detect PR from current branch
/pr-review 87           # Review specific PR number
```

## Pipeline

```
Architect (R1) → 10x GSD (R1) → Architect (R2) → 10x GSD (R2) → Security → Manager → [You Choose Fixes] → Senior Engineer
```

## Process

Read and follow the orchestrator instructions exactly:

**Orchestrator file:** `agents/review-orchestrator.md` (relative to this plugin's root)

The orchestrator will:
1. Verify `gh` CLI is installed and authenticated
2. Resolve which PR to review (auto-detect or user-specified)
3. Auto-detect languages and frameworks from the PR diff
4. Spawn reviewers sequentially with appropriate rule files
5. Present the manager's findings for interactive fix selection
6. Dispatch the senior engineer to implement selected fixes

**IMPORTANT:** Read the orchestrator file using the path `${CLAUDE_PLUGIN_ROOT}/agents/review-orchestrator.md` and follow it exactly.
