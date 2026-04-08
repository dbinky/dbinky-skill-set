---
name: auto-ralph-prep
description: Automatically generate ralph loop files (RALPH.md, focus-areas.md, gaps-identified.md) from spec and design docs without interactive Q&A
---

# Auto Ralph Prep

You are generating ralph-o-matic loop files without user interaction. This skill derives all configuration from the feature spec, design docs, implementation plans, and codebase scan — replacing the interactive Q&A in `plan-to-ralph`.

This is invoked either directly (`/auto-ralph-prep`) or via `plan-to-ralph --auto`.

## Arguments

Parse the following from the user's command or calling skill:

- `SPEC_PATH` (required): Path to the feature spec (e.g., `docs/specs/user-auth-spec.md`)
- `DESIGN_GLOB` (optional): Glob for design docs (default: auto-derived from spec path by replacing `-spec.md` with `-design-phase-*.md` in the superpowers specs directory)
- `PLAN_GLOB` (optional): Glob for implementation plans (default: auto-derived similarly)
- `--slug SLUG` (optional): Feature slug for commit messages (default: derived from spec filename)

## Prerequisites

Verify git repository:

```bash
git rev-parse --is-inside-work-tree
```

Verify spec file exists. If missing, stop with an error.

## Phase 1: Backup

Check if any of these files exist:
- `RALPH.md`
- `docs/reference/focus-areas.md`
- `docs/reference/gaps-identified.md`

If any exist, back them up using the same pattern as `plan-to-ralph`:

1. Create `docs/reference/historical/` if needed
2. Copy each existing file to `YYYY-MM-DD-historical-{filename}.md`
3. Append counter `-2`, `-3` for same-day collisions

## Phase 2: Read Inputs

Read these files in full:

1. The spec at `SPEC_PATH`
2. All design docs matching `DESIGN_GLOB`
3. All implementation plans matching `PLAN_GLOB`
4. `CLAUDE.md` (if it exists) for project constraints

## Phase 3: Codebase Scan

Use the Agent tool with `subagent_type: "Explore"` to discover codebase structure. Same scan prompt as `plan-to-ralph` Phase 2, with the spec's purpose as CONTEXT.

## Phase 4: Auto-Derive Configuration

Derive all 7 values that `plan-to-ralph` normally asks interactively:

### MISSION

Extract from the spec's purpose/goal/overview section. Format as:

> Review and refine the implementation of {feature description} per the spec at `{SPEC_PATH}`. The loop should verify correctness against the spec, ensure test coverage, fix gaps, and polish the implementation to production quality.

### TEST_COMMAND

Auto-detect from project files:
- `Makefile` with `test` target -> `make test`
- `package.json` with `test` script -> `npm test`
- `go.mod` -> `go test ./...`
- `pyproject.toml` -> `pytest tests/ -v`
- `Cargo.toml` -> `cargo test`

If nothing detected, default to `make test`.

### PERSONA

Generate based on the spec's domain, the project's tech stack, and the codebase patterns:

> You are a senior {LANGUAGE} engineer specializing in {DOMAIN_FROM_SPEC}. You understand {ARCHITECTURE_PATTERNS} and care deeply about {QUALITY_ATTRIBUTES_FROM_SPEC}. You review code with the rigor of a principal engineer preparing for a production deployment.

### SINGLE_AREAS

Derive from THREE sources, generating a comprehensive list:

**From implementation plan tasks:** Each plan task file becomes a focus area (Component category).

**From domain analysis of the spec:** Identify domain concepts that span multiple files — business entities, workflows, cross-cutting concerns. Each becomes a focus area (Domain Concept category).

**From vertical slice analysis:** Trace end-to-end paths through the system described in the spec — API endpoint -> business logic -> data layer -> response. Each becomes a focus area (Vertical Slice category).

Aim for **as many single focus areas as needed for deep coverage of new, impacted, and adjacent code**. Each gets 2 review passes, so 20 areas = 40 passes. Consolidate related files into a single area rather than listing each file separately — a focus area should be a logical unit of review, not a single file. Fewer, broader areas are better than many narrow ones.

### PAIRED_AREAS

Derive from cross-references between focus areas:

1. **Component Integration** — Two plan tasks that share models, APIs, or data structures
2. **Domain Boundary** — Two domain concepts that neighbor each other
3. **Vertical Slice Handoffs** — A slice paired with a component it passes through
4. **Cross-Cutting Consistency** — A cross-cutting concern paired with a specific component

Each paired area gets 1 review pass. Aim for **sufficient coverage to assure boundary and call pattern correctness** — focus on the highest-risk integration seams rather than exhaustively pairing everything.

### CHECKLIST

Always include these 3 universal items:
1. All tests pass (`{TEST_COMMAND}`)
2. No open issues in `docs/reference/gaps-identified.md` for this focus area
3. The focus area is complete and polished — you'd be proud to ship it

Add items derived from:
- Spec acceptance criteria -> "Code aligns with the spec at `{SPEC_PATH}`"
- Design doc existence -> "Implementation matches the design doc"
- Architecture patterns (if hexagonal/layered detected) -> "Module boundaries are clean"
- Security mentions in spec -> relevant security checklist item

### EXTRA_CONSTRAINTS

Pull from:
- `CLAUDE.md` constraints that apply to automated code work
- Any "constraints" or "non-goals" section in the spec

## Phase 5: Generate Files

Create `docs/reference/` directory if it doesn't exist.

**Read the example files for exact formatting reference before generating:**
- `${CLAUDE_PLUGIN_ROOT}/skills/plan-to-ralph/docs/reference/example-focus-areas.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/plan-to-ralph/docs/reference/example-gaps-identified.md`

Generate all 3 files using the exact same templates as `plan-to-ralph` Phase 4:

1. `RALPH.md` — using the MISSION, PERSONA, TEST_COMMAND, CHECKLIST, EXTRA_CONSTRAINTS
2. `docs/reference/focus-areas.md` — using SINGLE_AREAS and PAIRED_AREAS
3. `docs/reference/gaps-identified.md` — empty template

The templates are identical to those in the `plan-to-ralph` skill — reference that skill's Phase 4 for the exact markdown structure.

## Phase 6: Commit

```bash
git add RALPH.md docs/reference/focus-areas.md docs/reference/gaps-identified.md
git add docs/reference/historical/ 2>/dev/null || true
git commit -m "chore: generate ralph loop files for {SLUG}"
```

## Phase 7: Report

Output a brief summary (no user review needed):

```
Ralph loop files generated for {SLUG}:
  RALPH.md              — {PERSONA_FIRST_SENTENCE}
  focus-areas.md        — {N} single areas, {M} paired areas ({TOTAL} review passes)
  gaps-identified.md    — empty template

Files committed. Ready for /auto-ralph-submit or ralph-o-matic submit.
```

## Error Handling

On failure at any phase, send Teams notification via `ralph-o-matic notify` with error details and stop.
