---
name: auto-ralph-prep
description: Automatically generate ralph review files (RALPH.md, focus-areas.md, paired-reviews.md, gaps-identified.md) from spec, design and plan docs without interactive Q&A — behaviour-based areas, seam-based paired reviews, and an independent plan critique
---

# Auto Ralph Prep

You are generating ralph-o-matic review files without user interaction. This skill
derives the whole review plan from the feature spec, design docs, implementation plans
and a codebase scan — replacing the interactive Q&A in `plan-to-ralph`.

This is invoked either directly (`/auto-ralph-prep`) or via `plan-to-ralph --auto` or
the feature pipeline. The user is unavailable: make every call yourself.

Shared structure (read these when a phase says so):
- Templates: `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates/`
- Seeding rules: `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/seeding-rules.md`
- Plan critique: `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/plan-critique.md`
- Worked examples: `${CLAUDE_PLUGIN_ROOT}/skills/plan-to-ralph/docs/reference/example-*.md`

## Arguments

- `SPEC_PATH` (required): Path to the feature spec (e.g., `docs/specs/user-auth-spec.md`)
- `DESIGN_GLOB` (optional): Glob for design docs (default: derived from the spec path, `docs/superpowers/specs/{slug}-design-phase-*.md`)
- `PLAN_GLOB` (optional): Glob for implementation plans (default: `docs/superpowers/plans/{slug}-implementation-phase-*-task-*.md`)
- `--slug SLUG` (optional): Feature slug for commit messages (default: derived from the spec filename)
- `--max-iterations N` (optional): the iteration cap the run will use. If absent, the plan records a recommended cap.
- `IMPL_STATUS` (optional, from the feature pipeline): the draft implementation's STATUS block — `DONE` / `DONE_WITH_CONCERNS` / `BLOCKED`, remaining-task count, named blockers.

## Prerequisites

```bash
git rev-parse --is-inside-work-tree
```

Verify the spec file exists. If missing, stop with an error.

## Phase 1: Archive

Check for `RALPH.md`, `docs/reference/focus-areas.md`, `docs/reference/paired-reviews.md`,
`docs/reference/gaps-identified.md`. If any exist, archive them exactly as `ralph-plan`
Phase 2 does: **move (not copy)** each into `docs/reference/historical/` as
`YYYY-MM-DD-historical-{name}-{plan#}.md`, with the plan number = the highest existing
suffix for today plus 1 (not a file count).

**Why move instead of copy:** Phase 5 lays down fresh templates carrying the loop's
control semantics — role selection, the fix/verify separation and the `<promise>` tags
that let the ralph server stop. Moving the originals out guarantees a mangled
`RALPH.md` can never survive into the new run.

## Phase 2: Read Inputs

Read in full:

1. The spec at `SPEC_PATH`
2. All design docs matching `DESIGN_GLOB` — especially each phase's **Data Provenance** section
3. All implementation plans matching `PLAN_GLOB`
4. `CLAUDE.md` (if it exists) for project constraints
5. `docs/test-baseline.json` (if it exists) — the exact test commands and the pre-existing failing set
6. `git log --oneline $(git merge-base HEAD main)..HEAD` and the changed-file list

## Phase 3: Codebase Scan

Use the Agent tool with `subagent_type: "Explore"` to locate, for the feature described
in the spec: the composition root(s), the entry points (handlers, jobs, CLI commands,
boot steps, migrations) and sinks (table writes, publishes, responses, rendered
artifacts) the changed code sits between, and the test files covering each.

## Phase 4: Derive the Review Plan

Read `seeding-rules.md` in full. It is the authority for this phase.

### Test commands and baseline

- `TEST_COMMANDS`: the exact commands from `docs/test-baseline.json`. If the file is
  absent or its `suites` array is empty (a fallback baseline), auto-detect (`Makefile` test target → `make test`, `package.json` → `npm test`,
  `go.mod` → `go test ./...`, `*.sln`/`*.csproj` → `dotnet test`, `pyproject.toml` →
  `pytest`, `Cargo.toml` → `cargo test`) — every suite in a mixed-language repo.
- `BASELINE`: the pre-existing failing set from the baseline file, as test identifiers per
  suite (verifiers compare per test); if the file is absent or empty,
  `_(no baseline recorded — treat every failure as needing investigation)_`.

### Behaviours (single-system areas)

List what the spec **promises** — each capability, guarantee, removal or migration.
Plan tasks and design phases are evidence of where each promise is implemented, **not**
areas in their own right: a plan task that builds a repository, one that builds the
handler and one that wires them are three hops on one behaviour's path. For each
promise, find its entry and sink in the code and read the path. Apply the merge test,
then the split test (by sub-path, never by layer). Fold small changes in. Expect 6–15.

Rank by consequence (`high`/`med`/`low`), highest first. Build each `### Area N` block
from real greps — the design's Data Provenance table tells you which producer feeds
each input, and the wiring lens checks that it really does.

### Seams (paired reviews)

Apply the seam test to every pair of behaviours. The plans tell you which tasks built
which side, which is clause 2 of the test. One `A ↔ B` row per seam with a
`SHARED · TRACE · BREAK` cell; record every rejected combination under `## No seam`.

### Budget

`implied = 4 × singles + 2 × paired`. With `--max-iterations N`, require
`implied ≤ N × 0.35`, deferring the lowest-risk seams (then merging more) until it fits.
Without it, set `N = ceil(implied / 0.35)` as the recommended cap.

### Gaps to seed

Into `## Open` of `gaps-identified.md`, one `- [ ]` item each:
- real gaps found while seeding (an unfed input, a symbol constructed only in tests);
- any NEW test failure vs the baseline;
- if `IMPL_STATUS` is `DONE_WITH_CONCERNS` or `BLOCKED`, every named concern/blocker,
  with the plan task it came from. Put the behaviour it blocks first in risk order.

### Persona, mission, constraints

- `PERSONA`: "You are a senior {LANGUAGE} engineer specializing in {DOMAIN_FROM_SPEC}. You understand {ARCHITECTURE_PATTERNS} and care deeply about {QUALITY_ATTRIBUTES_FROM_SPEC}. You review code with the rigor of a principal engineer preparing for a production deployment."
- `MISSION`: 1–3 paragraphs — what was built per the spec at `{SPEC_PATH}`, what the review covers, and that done is baseline-relative and wired-and-fed.
- `EXTRA_CONSTRAINTS`: `CLAUDE.md` constraints that apply to automated code work, plus the spec's constraints and non-goals.

## Phase 5: Generate Files

Copy the templates **byte-exact** — never retype `RALPH.md`:

```bash
T="${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates"
mkdir -p docs/reference
cp "$T/RALPH.md"            RALPH.md
cp "$T/focus-areas.md"      docs/reference/focus-areas.md
cp "$T/paired-reviews.md"   docs/reference/paired-reviews.md
cp "$T/gaps-identified.md"  docs/reference/gaps-identified.md
```

Then, matching the worked examples (gap lines carry ids: `- [ ] G<n> **Title** — …`):
- write the behaviours (table + `### Area N` blocks) into `focus-areas.md`;
- write the seams and `## No seam` into `paired-reviews.md`;
- write the seeded gaps into `## Open` of `gaps-identified.md`;
- delete the templates' `<!-- … -->` example comments;
- replace each `{PLACEHOLDER}` in `RALPH.md` with Edit: `{PERSONA}`, `{MISSION}`,
  `{TEST_COMMANDS}` (one per line in a code block), `{BASELINE}`, `{DESIGN_SPECS}` (links
  to the spec, each design phase and the plan set), `{PLAN_ARITHMETIC}` (the exact shape
  from `seeding-rules.md`, including the `Iteration cap:` line), `{EXTRA_CONSTRAINTS}`.

**Never** write a checklist item requiring the whole suite to be green — done is NO NEW
failures vs the baseline and nothing that built at baseline broken.

## Phase 6: Independent Plan Critique

A plan reviewed only by its author keeps its author's slicing. Dispatch a **fresh
subagent** (Agent tool, `subagent_type: "general-purpose"`, `model: "opus"`) that did
not write the plan:

> You are the plan critic for a ralph review plan you did not write. Read
> `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/plan-critique.md` and
> `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/seeding-rules.md`, then apply the
> critique to `RALPH.md`, `docs/reference/focus-areas.md`,
> `docs/reference/paired-reviews.md` and `docs/reference/gaps-identified.md`. Spec:
> `{SPEC_PATH}`; designs: `{DESIGN_GLOB}`; plans: `{PLAN_GLOB}`; iteration cap: `{N}`.
> The seeder reported: {your summary of areas, seams, and which candidates you merged and
> why}. Read the code, not just the plan. Edit only those four files; never tick a box;
> never touch production code. Append the `## Plan critique` section to RALPH.md. Do not
> commit. Report areas and seams before/after, the implied passes and budget, and each
> merge or strike with its reason. The user is unavailable.

## Phase 7: Verify

```bash
for m in '## Role: wiring-review' '## Role: failure-review' '## Role: paired-review' \
         '## Role: verify' '## Wrap Up' '<promise>FINIT</promise>' \
         '<promise>CLOSER</promise>' 'Stop after the tag' '## Plan critique' \
         'Iteration cap:'; do
  grep -qF "$m" RALPH.md || echo "MISSING: $m"
done
grep -c '{[A-Z_]*}' RALPH.md     # must print 0 — every placeholder filled
grep -c '<!--' docs/reference/focus-areas.md docs/reference/paired-reviews.md docs/reference/gaps-identified.md   # 0 each
grep -n '\[[ x]\]' docs/reference/focus-areas.md | grep -v '^[0-9]*:|'   # must print nothing — no checkbox outside the table
```

If a RALPH.md marker is missing, the loop machinery was damaged: re-copy the template
and redo the RALPH.md part of Phase 5 (keep the plan files and the critique section).
Fix any other failure in place.

## Phase 8: Commit

```bash
git add RALPH.md docs/reference/focus-areas.md docs/reference/paired-reviews.md docs/reference/gaps-identified.md
git add docs/reference/historical/ 2>/dev/null || true
git commit -m "chore: generate ralph review plan for {SLUG}"
git push --quiet -u origin HEAD 2>/dev/null || true
```

## Phase 9: Report

```
Ralph review plan generated for {SLUG}:
  RALPH.md              — {PERSONA_FIRST_SENTENCE}
  focus-areas.md        — {S} behaviours (wiring + failure review each)
  paired-reviews.md     — {P} seams, {N} combinations recorded as no seam
  gaps-identified.md    — {G} seeded gaps
  Plan critique         — {areas A → B, seams C → D}
  Minimum passes        — {4S + 2P}; iteration cap {N}

Files committed. Ready for /auto-ralph-submit or ralph-o-matic submit --max-iterations {N}.
```

## Error Handling

On failure at any phase, send a Teams notification via `ralph-o-matic notify` with the
phase and error details, and stop.
