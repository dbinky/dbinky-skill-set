---
name: spec-to-design
description: Automated pipeline from spec+designs through alignment, planning, plan alignment, and draft implementation — runs steps 3-6 with zero interaction
---

# Spec to Design

You are running the automated implementation pipeline. Given a feature spec and its design docs, you will:

1. Align design docs to the spec
2. Produce detailed implementation plans
3. Align implementation plans to each other and the spec
4. Execute the implementation via parallel subagents

**This skill requires zero user interaction.** The user is unavailable. Do all work autonomously, committing at each phase.

## Arguments

Parse the following from the user's command or calling skill:

- `SPEC_PATH` (required): Path to the feature spec (e.g., `docs/specs/user-auth-spec.md`)
- `DESIGN_GLOB` (optional): Glob for design phase docs (default: derived from spec path — replace `-spec.md` with `-design-phase-*.md` in `docs/superpowers/specs/`)
- `PLAN_GLOB` (optional): Glob for plan docs (default: derived from spec path — replace `-spec.md` with `-implementation-phase-*-task-*.md` in `docs/superpowers/plans/`)
- `--slug SLUG` (optional): Feature slug for commit messages and file naming (default: derived from spec filename)

## Prerequisites

1. Verify `SPEC_PATH` exists. If not, notify Teams and stop.
2. Verify at least one file matches `DESIGN_GLOB`. If not, notify Teams and stop.
3. Read all input files to confirm they're valid markdown.

## Phase 1: Design Alignment

**Purpose:** Ensure design phase docs are coherent with the spec and with each other.

**Steps:**

1. Read the spec at `SPEC_PATH` thoroughly.
2. Read ALL design docs matching `DESIGN_GLOB`.
3. Review holistically for:
   - Alignment to the spirit of the spec (not just surface conformance)
   - Internal consistency across design phases (terminology, data models, use cases)
   - End-to-end coherence (does Phase 1's output feed correctly into Phase 2's input?)
   - Integration seams between phases
4. Fix any contradictions, inconsistencies, or misalignments directly in the design docs.
5. If no issues found, note that alignment is clean.
6. Commit:

```bash
git add docs/superpowers/specs/
git commit -m "docs: align {SLUG} design phases to spec"
```

**On failure:** Send Teams notification:
```bash
ralph-o-matic notify --message "Pipeline failed at design alignment for {SLUG}. Error: {ERROR}. Resume: /spec-to-design --spec {SPEC_PATH}"
```

## Phase 2: Implementation Plan Production

**Purpose:** Generate detailed, task-level implementation plans for each design phase.

**Steps:**

1. For each design phase doc, invoke the `superpowers:writing-plans` skill.
2. Each design phase may produce multiple task files.
3. Output naming: `docs/superpowers/plans/{SLUG}-implementation-phase-{N}-task-{M}.md`
   - N = design phase number
   - M = task number within that phase
4. Write ALL plans in one concerted effort — do not stop between phases.
5. Commit:

```bash
git add docs/superpowers/plans/
git commit -m "docs: write {SLUG} implementation plans"
```

**On failure:** Send Teams notification:
```bash
ralph-o-matic notify --message "Pipeline failed at plan production for {SLUG}. Error: {ERROR}. Resume: invoke superpowers:writing-plans manually for remaining design phases."
```

## Phase 3: Plan Alignment

**Purpose:** Ensure implementation plans are coherent with the spec, designs, and each other.

**Steps:**

1. Read the spec at `SPEC_PATH`.
2. Read ALL design docs matching `DESIGN_GLOB`.
3. Read ALL plan docs matching `PLAN_GLOB`.
4. Review holistically for:
   - Alignment to the spec's intent and the design docs
   - Internal consistency across all plan tasks (shared types, function signatures, API contracts)
   - Dependency ordering (does Task 2 depend on types defined in Task 1?)
   - No duplicate work across tasks
   - No gaps (every design requirement has a corresponding plan task)
5. Fix contradictions in the plan docs directly.
6. Commit:

```bash
git add docs/superpowers/plans/ docs/superpowers/specs/
git commit -m "docs: align {SLUG} implementation plans"
```

**On failure:** Send Teams notification:
```bash
ralph-o-matic notify --message "Pipeline failed at plan alignment for {SLUG}. Error: {ERROR}. Resume: /spec-to-design --spec {SPEC_PATH} (will re-run from alignment)"
```

## Phase 4: Draft Implementation

**Purpose:** Execute all implementation plans via parallel subagents.

**Steps:**

1. Read ALL plan docs matching `PLAN_GLOB`.
2. Analyze dependencies between phases and tasks:
   - Identify which tasks can run in parallel (no shared files, no dependency)
   - Identify which tasks must be sequential (Task B depends on Task A's output)
3. Invoke `superpowers:subagent-driven-development` to execute the plans:
   - Spawn a fresh subagent for each task
   - Each subagent uses `superpowers:executing-plans` with its task file
   - Parallelize independent tasks, sequence dependent ones
   - The user is unavailable — do not stop for review or permissions
4. After all subagents complete, run the full test suite:

```bash
make test
```

5. If tests fail, investigate and fix. Run tests again until they pass.
6. Commit all implementation:

```bash
git add -A
git commit -m "feat: implement {SLUG} via automated pipeline"
```

**On failure:** Send Teams notification:
```bash
ralph-o-matic notify --message "Pipeline failed at implementation for {SLUG}. Error: {ERROR}. Artifacts so far are committed. Resume manually or re-run /spec-to-design."
```

## Completion

After all 4 phases succeed, output:

```
spec-to-design complete for {SLUG}:
  Phase 1: Design alignment     ✓
  Phase 2: Plan production       ✓ ({N} plans across {M} phases)
  Phase 3: Plan alignment        ✓
  Phase 4: Implementation        ✓ (tests passing)

All code committed. Ready for /auto-ralph-prep.
```
