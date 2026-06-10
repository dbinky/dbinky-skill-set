---
name: auto-ralph-prep
description: Automatically generate ralph review files (RALPH.md, focus-areas.md, gaps-identified.md) from spec and design docs without interactive Q&A
---

# Auto Ralph Prep

You are generating ralph-o-matic review files without user interaction. This skill derives all configuration from the feature spec, design docs, implementation plans, and codebase scan — replacing the interactive Q&A in `plan-to-ralph`.

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

## Phase 1: Archive

Check if any of these files exist:
- `RALPH.md`
- `docs/reference/focus-areas.md`
- `docs/reference/gaps-identified.md`

**If none exist:** Skip to Phase 2.

**If any exist:** Archive them before regenerating, using the same pattern as `plan-to-ralph`:

1. Create `docs/reference/historical/` if it doesn't exist.
2. Get today's date as `YYYY-MM-DD`.
3. For each existing file, **move (rename) it** into `docs/reference/historical/` as `YYYY-MM-DD-historical-{filename}.md` — do NOT copy it. The original must no longer exist at its working-tree location afterward.
4. If a same-day backup already exists, append a counter: `-2`, `-3`, etc.

**Why move instead of copy:** Phase 5 regenerates each file fresh from the template embedded in this skill, which carries the required control semantics — notably the iteration/wrap-up protocol and the `<promise>` tag logic that lets the ralph server stop after repeated "no progress" iterations. Moving the originals out (rather than copying) guarantees they are absent when Phase 5 runs, so a previously mangled `RALPH.md` whose loop semantics were stripped can never survive into the new run. This subsumes the old legacy-format migration: any legacy or mangled `RALPH.md` is archived and replaced wholesale.

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

> Review and refine the implementation of {feature description} per the spec at `{SPEC_PATH}`. Verify correctness against the spec, ensure test coverage, fix gaps, and polish the implementation to production quality.

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

Generate all 3 files. The `RALPH.md` template is reproduced in full below — **write it verbatim, substituting only the `{VARIABLES}` from Phase 4.** For `focus-areas.md` and `gaps-identified.md`, use the same templates as `plan-to-ralph` Phase 4 (reference that skill for their exact structure).

> **CRITICAL — do not summarize or abridge `RALPH.md`.** The `MISSION`, `PERSONA`, `TEST_COMMAND`, `CHECKLIST`, and `EXTRA_CONSTRAINTS` values you derived in Phase 4 are *only the variable substitutions*. They are NOT the whole file. `RALPH.md` also contains fixed control machinery — the `## Tracking System`, `## Steps`, `## Wrap Up`, and `<promise>` tag sections — that drives the iteration loop and lets the ralph server stop after repeated "no progress" passes. If you generate `RALPH.md` from just the five variable sections, you will strip the loop semantics and the run will never terminate correctly. Reproduce every section shown below.

### Generate RALPH.md

Write `RALPH.md` in the repository root with this exact structure (substitute only the `{VARIABLES}`; keep every other section verbatim):

```markdown
# Review Instructions

You are an automated code reviewer. The user is unavailable — do the work without asking for input.

## Persona

{PERSONA}

## Your Mission

{MISSION_DESCRIPTION — expand the MISSION into 1-3 paragraphs describing what was built, what the review covers, and what "done" looks like}

## Tracking System

- Read `docs/reference/focus-areas.md` before starting. Each single area (from table 1) needs **2 review passes** before being considered __done__. Each paired area (from table 2) needs **1 review pass** before being considered __done__. Use this document to track which reviews have been advanced and completed.
- DO NOT update this document until the "Wrap Up" phase. Updating is conditional on your checklist assessment.

## Constraints

- **Do NOT use sub-agents for bulk generation.** When you modify or create code, do it by hand, one component at a time, with thought behind each decision.
- **Read before you write.** Before modifying any file, read the relevant sections. Before claiming something is fine, read it and reason about quality.
- **One focus area, one fix, one commit.** Pick a focus area. Find issues. Fix the ONE most important issue. Commit, push, and stop. Do not fix a second issue. Small, focused passes are more reliable than ambitious ones.
- **Do NOT invent new functionality to fill perceived gaps.** Maintain a list of things you find that should be fixed at `docs/reference/gaps-identified.md` in the `## Open Issues` section. If you perceive there is new, missing functionality beyond the current scope, log it in the `## Won't Fix (Beyond Current Scope)` section. If something on the list has been fixed previously, move it to `## Fixed Previously`.
{EXTRA_CONSTRAINTS — each as a new bullet point with bold lead, same format as above}

## Steps

Review one area, fix ONE issue, test, commit, stop. Resist the urge to keep fixing.

1. **Read the Tracking File** — Read `docs/reference/focus-areas.md` and pick a review focus area that hasn't been completed yet. Progression order: complete all **Pass 1** single area reviews, then **paired area** reviews, then **Pass 2** single area reviews.
2. **Read the Area's Code** — Deeply examine the code for the chosen focus area. Read every file. Understand the patterns.
3. **Analyze findings and update the gaps list** — Cross-reference what you just read with the design doc and codebase conventions. Add any issues found to `docs/reference/gaps-identified.md` in the `## Open Issues` section.
4. **Fix the single most important issue, then stop.** Fix it thoroughly — if it spans multiple files, fix all of them consistently. Once fixed, move the issue to `## Fixed Previously` in `docs/reference/gaps-identified.md`. **Proceed immediately to step 5. Do not fix another issue.**
5. **Run the tests** — Run `{TEST_COMMAND}`. Investigate and fix each failure.
6. **Assess the checklist** — Evaluate honestly, then proceed immediately to the Wrap Up phase. Do not go back to step 4.

## The Checklist (be brutally honest)

Do NOT check a box unless you could defend it in a code review:

{CHECKLIST — each item as `- [ ] {item text}`}

## Wrap Up

Follow these steps in order. **Do not go back to fix more issues.**

**Step A — Commit and push your branch to remote.**

Always commit and push first. Your work is valuable regardless of checklist status.

**Step B — Determine if this focus area passed review.**

All checklist boxes must be honestly checked for the focus area to pass. **Most focus areas need 2-5 passes — this is normal and expected.** A failing checklist simply means this area needs another pass. Every commit that fixes something is a successful outcome.

**Step C — Update tracking (only if the focus area passed).**

- If the focus area **passed**: Mark that focus area's review as complete in `docs/reference/focus-areas.md`.
- If the focus area **did not pass**: Do NOT update `docs/reference/focus-areas.md`.

**Step D — Output your promise tag and stop.**

Check `docs/reference/focus-areas.md`. Are ALL reviews (single areas and paired areas) now marked complete?

- If **all reviews are complete**: output `<promise>FINIT</promise>`
- If **any reviews remain incomplete**: output `<promise>CLOSER</promise>`

Output exactly one `<promise>` tag, then stop. Do not output anything after the tag.
```

### Generate the tracking files

2. `docs/reference/focus-areas.md` — using `SINGLE_AREAS` and `PAIRED_AREAS`, in the exact structure from `plan-to-ralph` Phase 4 (match `example-focus-areas.md`).
3. `docs/reference/gaps-identified.md` — empty template, exact structure from `plan-to-ralph` Phase 4 (match `example-gaps-identified.md`).

After writing `RALPH.md`, verify it contains the `## Steps`, `## Wrap Up`, and both `<promise>FINIT</promise>` / `<promise>CLOSER</promise>` tags before continuing. If any are missing, regenerate the file — it is incomplete.

## Phase 6: Commit

```bash
git add RALPH.md docs/reference/focus-areas.md docs/reference/gaps-identified.md
git add docs/reference/historical/ 2>/dev/null || true
git commit -m "chore: generate ralph review files for {SLUG}"
```

## Phase 7: Report

Output a brief summary (no user review needed):

```
Ralph review files generated for {SLUG}:
  RALPH.md              — {PERSONA_FIRST_SENTENCE}
  focus-areas.md        — {N} single areas, {M} paired areas ({TOTAL} review passes)
  gaps-identified.md    — empty template

Files committed. Ready for /auto-ralph-submit or ralph-o-matic submit.
```

## Error Handling

On failure at any phase, send Teams notification via `ralph-o-matic notify` with error details and stop.
