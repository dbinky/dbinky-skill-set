# Curating Code Skill Design

## Overview

`/curating-code [path]` analyzes a codebase (or targeted path) for vibe-coding cruft and transforms it into clean, idiomatic, senior-engineer-quality code. It runs four parallel specialist agents, consolidates their findings into a prioritized report, lets the user select which issues to address, and dispatches an implementer to execute the fixes.

## Problem Statement

Vibe-coded projects accumulate technical debt in predictable patterns: dead code from abandoned "fix forward" approaches, excessive complexity from AI-generated over-abstraction, copy-paste duplication, and inconsistent patterns from solving the same problem different ways across sessions. These issues compound over time, making codebases harder to maintain and reason about.

## Scope

- **Default**: whole codebase sweep from project root
- **Targeted**: accepts an optional path argument to narrow analysis to a specific file, directory, or module
- **Language support**: Go, C#, TypeScript, JavaScript, SQL, Terraform, HTML (extensible via new rule files)

## Pipeline

### Phase 1: Pre-flight

- Verify git repository
- Check for uncommitted changes (warn but don't block)
- Confirm target path exists and is non-empty

### Phase 2: Scope & Detection

- Resolve target path (default: project root, respecting `.gitignore`)
- Build file inventory by language/type
- Detect languages and frameworks from file extensions and content
- Select applicable curation rule files from `rules/curation/`

### Phase 3: Parallel Analysis

Dispatch four specialist agents simultaneously. Each receives the scoped file list and applicable curation rule files.

#### Dead Code Hunter (`agents/dead-code-hunter.md`)

Finds code that serves no purpose:

- Unused functions, methods, classes, and variables
- Unreachable branches (dead `if`/`else`, impossible conditions)
- Orphaned files (no imports referencing them)
- Commented-out code blocks
- Unused imports and dependencies
- Stale feature flags and their guarded code
- Each finding includes a confidence rating: high (definitely dead) or medium (likely dead, needs verification)

#### Complexity Reducer (`agents/complexity-reducer.md`)

Identifies patterns that are more complex than necessary:

- Over-abstracted patterns (factory-of-factory, unnecessary wrapper layers)
- Deep nesting (4+ levels that could be flattened with early returns or guard clauses)
- Convoluted control flow (state machines that should be conditionals, callback chains that should be async/await)
- God functions/files (too many responsibilities)
- Unnecessary indirection (interfaces with single implementations, abstract classes with one child)
- Each finding includes a specific simplification proposal

#### Duplication Consolidator (`agents/duplication-consolidator.md`)

Finds repeated code that should be consolidated:

- Near-identical code blocks (>70% structural similarity)
- Repeated patterns warranting a shared utility or helper
- Copy-pasted error handling, validation, or data transformation
- Similar API call patterns that could use a shared client/wrapper
- Each finding includes a consolidation strategy (extract function, create utility, parameterize differences)

#### Consistency Enforcer (`agents/consistency-enforcer.md`)

Detects divergent approaches to the same problem:

- Same problem solved multiple ways (e.g., 3 HTTP clients, 2 state management approaches)
- Mixed naming conventions within the same layer
- Inconsistent file organization patterns
- Conflicting approaches to cross-cutting concerns (logging, configuration, DI, error handling)
- Each finding recommends which pattern to converge on (most idiomatic or most prevalent)

### Phase 4: Consolidation

The orchestrator merges all findings using these rules:

**Deduplication**: When multiple agents flag the same code block, merge into one finding with perspectives from each.

**Cross-referencing**: When the consistency enforcer identifies a target pattern and the duplication consolidator found copies, link them as a single "convergence initiative."

**Conflict resolution**: Dead code findings always win over other categories (no point refactoring dead code). If the complexity reducer wants to extract a function that the dead code hunter flagged as unused, the dead code finding takes priority.

**Impact scoring**: Combine severity x confidence x estimated effort into a priority score for ordering within tiers.

**Dependency ordering**: Flag when fix B depends on fix A (e.g., remove dead code before consolidating duplicates that reference it).

### Phase 5: Curation Report

Present categorized findings:

```
## Curation Report: [project-name]
Scanned: 142 files across 3 languages (TypeScript, Go, SQL)
Analysis: 4 specialists, 47 findings (12 deduplicated -> 35 unique)

### Critical (4 findings)
- [DC-001] `services/auth/legacy.ts` - Entire module unreferenced after OAuth2 migration (high confidence)
- [CX-003] `handlers/order.go` - 7-level nested conditional, can flatten to guard clauses (high confidence)
...

### Important (11 findings)
...

### Consider (14 findings)
...

### Cosmetic (6 findings)
...
```

**Tier definitions:**
- **Critical**: Actively harms maintainability, causes confusion, or hides bugs
- **Important**: Significant quality improvement, reduces cognitive load meaningfully
- **Consider**: Minor quality gain, worth doing if touching the area anyway
- **Cosmetic**: Style and consistency nits, lowest priority

### Phase 6: Interactive Selection

User selects which findings to address:
- Select entire tiers ("all critical", "all important")
- Select individual findings by ID
- Skip/defer specific findings
- "Select all" option for full cleanup

### Phase 7: Implementation

Implementer agent (`agents/curator-implementer.md`) executes selected fixes:

- Respects dependency ordering from consolidation phase
- Commits logical groups separately (one initiative per commit, not one giant commit)
- Every change must be behavior-preserving; flags semantic changes for manual review
- Runs existing test suite after each logical group if tests exist; if no tests exist, proceeds but notes this in the summary as a risk
- Reports findings it couldn't address automatically

**Final summary:**
```
## Curation Complete
- 18 findings addressed across 12 files
- 340 lines removed, 85 lines added (net -255)
- 3 commits created on current branch
- 2 findings deferred (require manual review)
```

## Finding Format

All agents produce findings in a consistent structure:

```
Finding ID: {category prefix}-{number}  (DC-, CX-, DU-, CN-)
Category: dead-code | complexity | duplication | consistency
Location: file:line_range
Severity: critical | important | consider | cosmetic
Confidence: high | medium
Description: What the issue is
Evidence: Why this is an issue (imports graph, call analysis, etc.)
Recommendation: Specific action to take
Estimated effort: trivial | small | moderate | large
```

## Agents

| Agent | File | Role |
|-------|------|------|
| Curation Orchestrator | `agents/curation-orchestrator.md` | Coordinates pipeline, detects languages, dispatches specialists, consolidates findings |
| Dead Code Hunter | `agents/dead-code-hunter.md` | Finds unused and unreachable code |
| Complexity Reducer | `agents/complexity-reducer.md` | Identifies over-complex patterns with specific simplifications |
| Duplication Consolidator | `agents/duplication-consolidator.md` | Finds repeated code with consolidation strategies |
| Consistency Enforcer | `agents/consistency-enforcer.md` | Detects divergent patterns with convergence recommendations |
| Curator Implementer | `agents/curator-implementer.md` | Executes selected fixes with behavior preservation |

## Curation Rules

Separate from pr-review rules. Located at `rules/curation/`. Each file covers:

- **Idiomatic Patterns**: What senior engineers write in this language
- **Vibe-Code Anti-Patterns**: Common patterns LLM-assisted coding produces (over-verbose, safety-net abstractions, redundant error handling)
- **Simplification Strategies**: Specific before/after transformations
- **Dead Code Signals**: Language-specific indicators of dead code

| Rule File | Language/Tool |
|-----------|--------------|
| `rules/curation/go.md` | Go (error handling, interface sizing, goroutine cleanup, channel patterns) |
| `rules/curation/csharp.md` | C# (LINQ, pattern matching, nullable refs, async/await, dispose) |
| `rules/curation/typescript.md` | TypeScript (type narrowing, discriminated unions, generics, module patterns) |
| `rules/curation/javascript.md` | JavaScript (optional chaining, nullish coalescing, Promise patterns) |
| `rules/curation/sql.md` | SQL (N+1 patterns, indexing, CTE vs subquery, join optimization) |
| `rules/curation/terraform.md` | Terraform (module composition, variable hygiene, state management) |
| `rules/curation/html.md` | HTML (semantic elements, accessibility, form patterns) |

## Plugin Registration

Add skill path to `.claude-plugin/plugin.json` under the `skills` array:
```json
"./skills/curating-code"
```

## File Structure (new files)

```
skills/
  curating-code/
    SKILL.md                          # User-facing skill entry point
agents/
  curation-orchestrator.md            # Pipeline coordinator
  dead-code-hunter.md                 # Specialist: dead/unused code
  complexity-reducer.md               # Specialist: over-complex patterns
  duplication-consolidator.md         # Specialist: repeated code
  consistency-enforcer.md             # Specialist: divergent patterns
  curator-implementer.md              # Implementation agent
rules/
  curation/
    go.md
    csharp.md
    typescript.md
    javascript.md
    sql.md
    terraform.md
    html.md
```
