---
name: curating-code
description: Analyze a codebase for vibe-coding cruft — dead code, excessive complexity, duplication, and inconsistent patterns — then selectively fix issues to produce clean, senior-engineer-quality code.
---

# Codebase Curation

Analyze a codebase (or targeted path) for vibe-coding cruft using four parallel specialist agents. Consolidates findings into a prioritized report, lets you select which issues to address, and implements the fixes.

## Usage

```
/curating-code              # Sweep the entire codebase
/curating-code src/          # Curate a specific directory
/curating-code lib/auth.ts   # Curate a specific file
```

## Pipeline

```
Scope & Detect → [Dead Code Hunter | Complexity Reducer | Duplication Consolidator | Consistency Enforcer] → Consolidate → Report → Select → Implement
```

## Specialists

| Agent | Finding Prefix | Focus |
|-------|----------------|-------|
| Dead Code Hunter | `DC-` | Unused functions, unreachable branches, orphaned files, stale imports |
| Complexity Reducer | `CX-` | Over-abstraction, deep nesting, convoluted control flow, god functions |
| Duplication Consolidator | `DU-` | Near-identical blocks, copy-paste patterns, consolidation opportunities |
| Consistency Enforcer | `CN-` | Divergent patterns, mixed conventions, conflicting approaches |

## Process

Read and follow the orchestrator instructions exactly:

**Orchestrator file:** `agents/curation-orchestrator.md` (relative to this plugin's root)

The orchestrator will:
1. Verify git repository and resolve target scope
2. Detect languages and frameworks, load applicable curation rule files
3. Dispatch all four specialist agents in parallel
4. Consolidate findings: deduplicate, cross-reference, resolve conflicts, assign tiers
5. Present a categorized curation report (Critical / Important / Consider / Cosmetic)
6. Let you select which findings to address (by tier or individually)
7. Dispatch the curator implementer for selected fixes

**IMPORTANT:** Read the orchestrator file using the path `${CLAUDE_PLUGIN_ROOT}/agents/curation-orchestrator.md` and follow it exactly.
