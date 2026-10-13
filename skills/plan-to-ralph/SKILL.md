---
name: plan-to-ralph
description: Generate ralph-o-matic review files via interactive Q&A or automatic derivation (--auto) from spec and design docs — behaviour-based focus areas, seam-based paired reviews, and a role protocol that separates fixing from approving
---

# Plan to Ralph

You are generating ralph-o-matic review files through interactive Q&A. This skill produces four files that configure a ralph refinement run:

- `RALPH.md` — the charter: persona, mission, test commands, baseline, and the loop protocol (role selection, fix/verify separation, wrap-up, promise tags)
- `docs/reference/focus-areas.md` — single-system reviews: one row per **behaviour**, two approvals each (wiring lens, failure lens), one detail block per area
- `docs/reference/paired-reviews.md` — paired reviews: one row per **seam**, one approval covering both directions, plus `## No seam`
- `docs/reference/gaps-identified.md` — open gaps, escalations, closed, won't fix

The structure comes from the `ralph-plan` skill's templates and reference docs, which this skill shares:
- Templates: `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates/`
- Seeding rules: `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/seeding-rules.md`
- Plan critique: `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/plan-critique.md`

After generating these files, the user runs `/direct-to-ralph` or `ralph-o-matic submit` to start the review.

## Arguments

Parse the following from the user's command:

- `CONTEXT` (optional): Free-text description of what the review should focus on (e.g., "on the work on the new identity system"). When provided, narrows codebase scan and pre-seeds Q&A answers.
- `--reset`: Skip backup. Overwrite existing files without creating historical copies.
- `--max-iterations N` (optional): the iteration cap the run will use; skips Question 6.
- `--auto`: Skip all Q&A and derive answers automatically from spec and design docs. Requires `SPEC_PATH` argument (path to the feature spec). When `--auto` is passed, this skill delegates entirely to the `auto-ralph-prep` skill. Equivalent to invoking `/auto-ralph-prep` directly.
- `SPEC_PATH` (required when `--auto`): Path to the feature spec (e.g., `docs/specs/user-auth-spec.md`)

## Prerequisites

Before starting, verify:

```bash
git rev-parse --is-inside-work-tree
```

If this fails, stop and tell the user: "This skill requires a git repository. Initialize one with `git init` first."

---

## Auto Mode Gate

**If `--auto` was passed:**

Invoke the `auto-ralph-prep` skill with the same arguments:
- Pass `SPEC_PATH` as the spec path
- Pass `CONTEXT` (if provided) as the design glob context
- Pass `--max-iterations` if provided
- Pass `--slug` derived from the spec filename

Do NOT continue with the phases below — `auto-ralph-prep` handles everything.

**If `--auto` was NOT passed:** Continue with Phase 1 below (interactive Q&A flow).

---

## Phase 1: Backup

**Skip this phase if `--reset` was passed.**

Check if any of these files exist:
- `RALPH.md`
- `docs/reference/focus-areas.md`
- `docs/reference/paired-reviews.md`
- `docs/reference/gaps-identified.md`

**If none exist:** Skip silently to Phase 2.

**If any exist:** Archive them exactly as `ralph-plan` Phase 2 does — **move (not copy)**
each into `docs/reference/historical/` as `YYYY-MM-DD-historical-{name}-{plan#}.md`,
where the plan number is the **highest existing suffix for today plus 1** (not a file
count — a partial earlier backup makes count-based numbering collide). Tell the user:
"Archived existing files to `docs/reference/historical/` as plan #{plan#}."

**Why move instead of copy:** Phase 4 lays down fresh templates carrying the loop's
control semantics — role selection, the fix/verify separation and the `<promise>` tag
logic that lets the ralph server stop. Moving the originals out guarantees a previously
mangled `RALPH.md` can never survive into the new run.

---

## Phase 2: Scan

Use the Agent tool with `subagent_type: "Explore"` to discover codebase structure. Send the subagent this prompt:

> Explore this codebase and report back a structured inventory. I need:
>
> 1. **Project type** — What language(s), framework(s), build system(s)? Look for: go.mod, package.json, pyproject.toml, Cargo.toml, *.csproj, *.sln, Makefile, CMakeLists.txt, etc.
>
> 2. **Components** — Group files by directory/module. For each component, report:
>    - Name (directory name or module name)
>    - Key files (entry points, main logic files)
>    - One-line description of what it does
>
>    Look in these standard locations: `src/`, `internal/`, `lib/`, `cmd/`, `app/`, `packages/`, `pkg/`, `modules/`, and top-level directories that contain source code.
>
> 3. **Entry points** — Files like main.go, main.py, index.ts, Program.cs, app.py, server.ts, Dockerfile
>
> 4. **Config/infra** — CI/CD workflows (.github/workflows/), docker files, migration directories, config directories
>
> 5. **Test directories** — Map test files/directories back to the components they test. Look for: tests/, test/, *_test.go, *.test.ts, test_*.py, *.spec.ts, *_test.py
>
> 6. **Design docs or specs** — Any files in docs/plans/, docs/specs/, docs/designs/, docs/reference/ that describe the system
>
> {IF CONTEXT WAS PROVIDED: "7. **Context focus** — The user wants to focus on: '{CONTEXT}'. Identify which components are directly related to this context, and which components touch those (same package, co-located files, shared config, related tests). Mark each component as 'direct' or 'indirect'."}
>
> Return the results as a structured list grouped by category.

Store the scan results for use in Phase 3.

**If the scan finds zero components** (empty repo or no source code), note this — you'll ask the user to define focus areas manually in Phase 3.

---

## Phase 3: Q&A

Ask questions **one at a time**. Skip questions that CONTEXT already answers.

### Question 1: Mission

**Skip if CONTEXT provides a clear description of what to review.**

Ask: "What should this ralph review improve? Describe the feature, system, or area of code."

Store the answer as `MISSION`.

If CONTEXT was provided, use it as the MISSION and tell the user: "Based on your context, the review will focus on: {CONTEXT}. Sound right?" Accept corrections.

### Question 2: Test Command

Auto-detect the test command by checking for:
- `Makefile` → look for `test` target (use `make test`)
- `package.json` → look for `test` script (use `npm test`)
- `pyproject.toml` → (use `pytest tests/ -v`)
- `go.mod` → (use `go test ./...`)
- `Cargo.toml` → (use `cargo test`)
- `*.sln` or `*.csproj` → (use `dotnet test`)

Present the detected command: "I detected `{COMMAND}` as your test command. Use this, or enter a different one?"

If nothing is detected, ask: "What command runs your tests? (e.g., `make test`, `npm test`, `pytest`)"

The test command is required. If the repo is mixed-language, collect every suite's command. Store it as `TEST_COMMAND`.

### Question 3: Persona

Generate a persona paragraph based on:
- The language/framework from the scan
- The MISSION description
- Any design docs found in the scan
- The architecture patterns visible in the codebase (e.g., hexagonal, MVC, microservices)

The persona should follow this pattern (drawn from real examples):
> You are a [ROLE] who [RELEVANT EXPERTISE]. You understand [ARCHITECTURE/PATTERNS] and [DOMAIN KNOWLEDGE]. You care about [QUALITY ATTRIBUTES].

Present it and ask:
1. **Accept** — use this persona as-is
2. **See alternatives** — show 3 more persona suggestions (plus the original, so 4 total to choose from)
3. **Write your own** — type a custom persona

If they choose "See alternatives," generate 3 additional personas with different emphases (e.g., security-focused, test-coverage-focused, spec-alignment-focused, performance-focused) and present all 4 as a numbered list. They can pick a number or type their own.

Store the final persona as `PERSONA`.

### Question 4: Behaviours (single-system areas)

Read `seeding-rules.md` first; it is the authority for this question and the next.

**If the scan found components:** derive candidate **behaviours**, not components. For
each thing the work promises (from CONTEXT, the MISSION, the spec/designs, the commit
log), find its entry point and sink and read the path between them. Apply the merge
test to every pair and the split test (by sub-path, never by layer) to any candidate
too large for one pass. Fold small changes into the behaviour they serve. Expect 6–15;
above 20 means you are slicing by layer — merge.

Present them highest risk first:

```
Candidate behaviours (each gets two reviews: wiring lens, failure lens):

  [1] [high] a saved search is stored for its owner and refused for anyone else
        entry api/handlers/saved_search.go:Create → sink store/saved_search_repo.go:Insert
        folded in: route registration, migration 0042 constraints
  [2] [high] the nightly job emails each owner the new listings matching their saved searches
        entry jobs/alerts.go:RunNightly → sink mail/sender.go:Send
  [3] [med]  the unsubscribe link turns alerts off for exactly one saved search
        entry api/handlers/unsubscribe.go:Handle → sink store/saved_search_repo.go:SetEnabled

Merged while deriving: "saved-search handler", "saved-search repository" and
"filters value object" → [1] (one path; none judgeable without the others).

Keep, drop, merge, split (by sub-path), re-rank, or add?
```

**If the scan found zero components:** ask: "I couldn't detect clear components. Describe
what this work promises — one behaviour per line, ideally with where it starts and where
its effect lands."

For each confirmed behaviour, build its `### Area N` block (Promise, Entry, Sink, Files,
Folded in, 3–7 wiring-lens checks, 3–7 failure-lens checks, Seed evidence) from real
greps. Store as `SINGLE_AREAS`.

### Question 5: Seams (paired reviews)

Apply the seam test to every pair of confirmed behaviours: shared state, contract,
queue, flag or ordering **and** different authoring tasks/commits **and** a silent
failure. One `A ↔ B` row per seam — never a reverse row.

```
Seams (one review each, both directions):

  [1] [high] saved search is stored ↔ nightly job emails matches
        SHARED: saved_searches.filters JSON shape · BREAK: matcher ignores a key the handler writes
  [2] [med]  unsubscribe ↔ nightly job
        SHARED: the enabled flag, disable-before-run ordering · BREAK: cached list emails an unsubscribed owner

Not paired (recorded under ## No seam):
  stored / unsubscribe — share the table, no shared field or ordering

Keep, drop, add, or re-rank?
```

Store the rows as `PAIRED_AREAS` and the rejections as `NO_SEAM`.

### Question 6: Iteration cap

**Skip if `--max-iterations` was passed.**

Compute the implied minimum `4 × singles + 2 × paired` (every box costs a refine pass and
a separate verify pass) and the recommended cap `ceil(implied / 0.35)`:

"This plan needs at least {implied} passes; with headroom for gaps and regressions I
recommend `--max-iterations {recommended}`. Use that, or tell me the cap you'll run with?"

If the user's cap is below the recommendation, defer the lowest-risk seams to
`Deferred (budget)` (then merge more) until `implied ≤ cap × 0.35`, and show what moved.
Store as `MAX_ITERATIONS`.

### Question 7: Additional Constraints

Present the standard constraints that are always included (they are part of `RALPH.md`):

```
These constraints are always included:
- No sub-agents; read before you write
- One item per pass; fix the most important issue, then stop
- The pass that fixes never approves — a later verify pass ticks the box
- Don't invent new functionality (log gaps instead)
- Tests run in the foreground; done is baseline-relative, never "whole suite green"

Any project-specific constraints to add? (e.g., "Do NOT write Python scripts for analysis", "Run linting after every change"). Enter to skip.
```

Store any additions as `EXTRA_CONSTRAINTS`.

---

## Phase 4: Generate

### 4.1 — Copy the templates (byte-exact)

```bash
T="${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates"
mkdir -p docs/reference
cp "$T/RALPH.md"            RALPH.md
cp "$T/focus-areas.md"      docs/reference/focus-areas.md
cp "$T/paired-reviews.md"   docs/reference/paired-reviews.md
cp "$T/gaps-identified.md"  docs/reference/gaps-identified.md
```

Never retype `RALPH.md` — a shell copy is how its loop semantics survive intact.

### 4.2 — Fill the plan files

Match the worked examples exactly:
- `${CLAUDE_PLUGIN_ROOT}/skills/plan-to-ralph/docs/reference/example-focus-areas.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/plan-to-ralph/docs/reference/example-paired-reviews.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/plan-to-ralph/docs/reference/example-gaps-identified.md`

Write `SINGLE_AREAS` (table + blocks) into `focus-areas.md`, `PAIRED_AREAS` and `NO_SEAM`
into `paired-reviews.md`, and any gaps found while seeding into the `## Open` section of
`gaps-identified.md`. Delete the templates' `<!-- … -->` example comments.

### 4.3 — Critique the plan

Apply `plan-critique.md` to what you just wrote (merge, block, seam, budget and risk-order
tests, from outside). Fix what fails, renumber, and append the `## Plan critique` section
to `RALPH.md`.

### 4.4 — Fill RALPH.md placeholders

With Edit (not a rewrite), replace:
- `{PERSONA}` — the PERSONA from Question 3
- `{MISSION}` — the MISSION expanded into 1–3 paragraphs: what was built, what the review covers, what done looks like
- `{TEST_COMMANDS}` — TEST_COMMAND (and any other suites), one per line in a code block
- `{BASELINE}` — the pre-existing failing tests if `docs/test-baseline.json` exists or the user supplied them; otherwise `_(no baseline recorded — treat every failure as needing investigation)_`
- `{DESIGN_SPECS}` — links to the spec/design/plan docs found in the scan
- `{PLAN_ARITHMETIC}` — singles, paired, implied minimum, MAX_ITERATIONS, budget (`MAX_ITERATIONS × 0.35`)
- `{EXTRA_CONSTRAINTS}` — bold-lead bullets, or `_(none)_`

### 4.5 — Verify

```bash
for m in '## Role: wiring-review' '## Role: failure-review' '## Role: paired-review' \
         '## Role: verify' '## Wrap Up' '<promise>FINIT</promise>' \
         '<promise>CLOSER</promise>' 'Stop after the tag' '## Plan critique' \
         'Iteration cap:'; do
  grep -qF "$m" RALPH.md || echo "MISSING: $m"
done
grep -c '{[A-Z_]*}' RALPH.md            # must print 0
grep -c '<!--' docs/reference/focus-areas.md docs/reference/paired-reviews.md docs/reference/gaps-identified.md   # 0 each
```

Anything missing → re-copy the template and redo 4.4.

---

## Phase 5: Review

Show the user a summary:

```
Ralph review files generated:

  RALPH.md
    Persona:       {first sentence of PERSONA}
    Test commands: {TEST_COMMANDS}
    Plan critique: {before → after line}

  docs/reference/focus-areas.md     — {S} behaviours (wiring + failure review each)
  docs/reference/paired-reviews.md  — {P} seams, {N} combinations recorded as no seam
  docs/reference/gaps-identified.md — {G} seeded gaps

  Minimum passes: {4S + 2P}   Cap: {MAX_ITERATIONS}

Want to review or edit any of the generated files before I commit?
```

If they want to review a file, show its content using the Read tool. If they make manual edits (outside Claude Code), re-read the file to acknowledge changes. If no review needed, proceed to Phase 6.

---

## Phase 6: Commit

Stage and commit all generated files plus any historical backups:

```bash
git add RALPH.md docs/reference/focus-areas.md docs/reference/paired-reviews.md docs/reference/gaps-identified.md
git add docs/reference/historical/ 2>/dev/null || true
git commit -m "chore: generate ralph review files via plan-to-ralph"
```

Tell the user:

```
Review files committed. To start the ralph review:

  /direct-to-ralph "{MISSION_SHORT}" --max-iterations {MAX_ITERATIONS}

Or submit manually:

  ralph-o-matic submit --max-iterations {MAX_ITERATIONS}
```
