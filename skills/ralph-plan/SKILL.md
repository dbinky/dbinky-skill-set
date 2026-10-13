---
name: ralph-plan
description: Automate ralph review setup — archives existing tracking docs, copies templates, seeds behaviour-based focus areas and seam-based paired reviews from branch changes or user-specified scope, critiques the plan, and links design docs into RALPH.md
---

# Ralph Plan

Prepare a project for a ralph refinement run: archive the old tracking documents, lay
down fresh templates, and seed a review plan of **behaviours** (single-system areas,
two lenses each) and **seams** (paired reviews, one row per seam) from the work scope.

## Usage

```
/ralph-plan                           # Scope = changes in current branch vs main
/ralph-plan PR #123                   # Scope = changes in a specific PR
/ralph-plan "the new auth system"     # Scope = user-described body of work
/ralph-plan --max-iterations 300      # Size the plan to a known iteration cap
```

## Process

Follow each phase in order. Do not skip phases.

Reference docs used below (read them when the phase says so):
- `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/seeding-rules.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/plan-critique.md`

---

## Phase 1: Define Scope

Parse the user's arguments:

- **No arguments:** Scope is changes in the current branch compared to the default branch. Run:
  ```bash
  git log --oneline $(git merge-base HEAD main)..HEAD
  git diff --name-only $(git merge-base HEAD main)..HEAD
  ```
  If no divergence from main (i.e., we ARE on main or no commits ahead), ask: "You're on the default branch with no divergent commits. What should the scope be?"

- **PR number (e.g., `#123` or `123`):** Fetch the PR diff:
  ```bash
  gh pr diff 123
  gh pr view 123 --json title,body
  ```

- **Free text description:** Use the description as scope context. Still run `git diff --name-only` against main to find changed files if any exist.

- **`--max-iterations N`** (optional): the iteration cap the run will be submitted with. Store as `MAX_ITERATIONS`. If absent, Phase 5 recommends one from the plan's size.

Store the result as `SCOPE` — a description of what's being reviewed plus the list of changed files.

---

## Phase 2: Archive Previous Documents

Check for existing files:
- `RALPH.md`
- `docs/reference/focus-areas.md`
- `docs/reference/paired-reviews.md`
- `docs/reference/gaps-identified.md`

**If none exist:** Skip to Phase 3.

**If any exist:**

1. Create `docs/reference/historical/` and determine today's date as `YYYY-MM-DD`.
2. Determine the plan number from the **highest existing suffix for today**, plus 1 —
   NOT a file count. A partial earlier backup (one of three files) makes count-based
   numbering collide with an earlier set:
   ```bash
   last=$(ls docs/reference/historical/YYYY-MM-DD-historical-*-[0-9]*.md 2>/dev/null \
     | sed -E 's/.*-([0-9]+)\.md$/\1/' | sort -n | tail -1)
   echo $(( ${last:-0} + 1 ))
   ```
3. **Move (rename) — do NOT copy —** each existing file into
   `docs/reference/historical/` as `YYYY-MM-DD-historical-{name}-{plan#}.md`
   (`{name}` = `RALPH`, `focus-areas`, `paired-reviews`, `gaps-identified`). The
   original must no longer exist at its working-tree location afterward.
4. Report: "Archived existing tracking docs as plan #{plan#} for {date}."

**Why move instead of copy:** Phase 3 lays down fresh templates that carry the required
control semantics — the role selection, the fix/verify separation, and the `<promise>`
tag logic that lets the ralph server stop. Moving the originals out guarantees they are
absent in Phase 3, so a previously mangled `RALPH.md` can never survive into the new run.

---

## Phase 3: Copy Templates

Copy the fresh templates **with `cp`** (a shell copy is byte-exact; retyping a template
is how loop semantics get dropped):

```bash
T="${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates"
mkdir -p docs/reference
cp "$T/RALPH.md"            RALPH.md
cp "$T/focus-areas.md"      docs/reference/focus-areas.md
cp "$T/paired-reviews.md"   docs/reference/paired-reviews.md
cp "$T/gaps-identified.md"  docs/reference/gaps-identified.md
```

**Always copy `RALPH.md` from the template — never preserve an existing one.** Anything
worth keeping from an old file lives in its archived copy; later phases re-apply the
project-specific sections.

---

## Phase 4: Detect Test Commands and Baseline

1. Detect the project's test command(s) — the repo may be mixed-language, so there may
   be more than one. If `docs/test-baseline.json` exists, use its exact commands.
   Otherwise check `CLAUDE.md`, `README`, `Makefile`, and the build manifests
   (`go.mod` → `go test ./...`, `package.json` → `npm test`, `*.sln`/`*.csproj` →
   `dotnet test`, `pyproject.toml` → `pytest`, `Cargo.toml` → `cargo test`).
2. Baseline: if `docs/test-baseline.json` exists, summarise its pre-existing failing
   set. Otherwise run each command once **in the foreground** (never end the turn while
   a suite is running) and record which tests already fail — that is the baseline.
   If you cannot run them, say so: `_(no baseline recorded — treat every failure as
   needing investigation)_`.

---

## Phase 5: Seed the Review Plan

Read `seeding-rules.md` in full first. It is the authority for everything in this phase.

1. **Find the behaviours** in `SCOPE`: list what the work promises; for each promise,
   find its entry point and sink and read the path between them. Apply the merge test
   to every pair of candidates and the split test (by sub-path, never by layer) to any
   that is too large. Fold small changes into the behaviour they serve.
2. **Fill `docs/reference/focus-areas.md`:** the five-column table, highest risk first,
   and one `### Area N` block per row with real `file:symbol` evidence from the greps
   you run now. Delete the template's example comments.
3. **Find the seams** and fill `docs/reference/paired-reviews.md`: apply the seam test,
   one `A ↔ B` row per seam with a `SHARED · TRACE · BREAK` cell, no reverse rows, and
   the `## No seam` section for every combination you considered and rejected.
4. **Seed `docs/reference/gaps-identified.md`** with the real gaps you found while
   seeding.
5. **Budget.** Compute the implied minimum `4 × singles + 2 × paired`.
   - With `MAX_ITERATIONS` known: the plan must satisfy `implied ≤ MAX_ITERATIONS × 0.35`.
     If it does not, defer the lowest-risk seams to `Deferred (budget)`, then merge more.
   - Without it: recommend `ceil(implied / 0.35)` as the iteration cap.

Present the plan to the user:

```
Behaviours (single-system, two lenses each):
  1. [high] <name> — entry <file:symbol> → sink <file:symbol>
  ...
Seams (one row each, both directions):
  1. [high] <A> ↔ <B> — BREAK: <…>
  ...
Not paired: {N} combinations recorded under ## No seam
Budget: {singles} singles, {paired} seams → minimum {implied} passes; cap {MAX_ITERATIONS or "recommend N"}
```

Ask: "Want to add, remove, merge or adjust any behaviours or seams?" Apply their
feedback (re-running the merge and seam tests on anything they add), then write the
final files.

---

## Phase 6: Critique the Plan

Read `plan-critique.md` and apply it to the plan you just wrote: the merge, block, seam,
budget and risk-order tests, applied from outside. Fix what fails, renumber, and append
the `## Plan critique` section to `RALPH.md`. Tell the user what changed (or that the
plan was already tight).

---

## Phase 7: Fill RALPH.md

Replace each `{PLACEHOLDER}` in `RALPH.md` with an Edit — do not retype the file:

- `{PERSONA}` — a persona for this codebase and mission (role, expertise, architecture
  patterns, quality bar).
- `{MISSION}` — 1–3 paragraphs: what was built, what the review covers, what done looks like.
- `{TEST_COMMANDS}` — the commands from Phase 4, one per line in a code block.
- `{BASELINE}` — the baseline summary from Phase 4.
- `{DESIGN_SPECS}` — links to every relevant document. Search `docs/specs/`,
  `docs/plans/`, `docs/designs/`, `docs/superpowers/specs/`, `docs/superpowers/plans/`,
  and any README or design doc referenced in changed files:
  ```markdown
  - `docs/superpowers/specs/some-design.md` — brief description
  ```
  If none: `_(no design docs found — consider creating a spec before running the review)_`
- `{PLAN_ARITHMETIC}` — singles, paired, implied minimum, cap (or recommended cap), budget.
- `{EXTRA_CONSTRAINTS}` — project constraints from `CLAUDE.md` that apply to automated
  code work, as bold-lead bullets; `_(none)_` if there are none.

Then verify the loop machinery survived:

```bash
for m in '## Role: wiring-review' '## Role: failure-review' '## Role: paired-review' \
         '## Role: verify' '## Wrap Up' '<promise>FINIT</promise>' \
         '<promise>CLOSER</promise>' 'Stop after the tag' '## Plan critique' \
         'Iteration cap:'; do
  grep -qF "$m" RALPH.md || echo "MISSING: $m"
done
grep -c '{[A-Z_]*}' RALPH.md   # must print 0 — every placeholder filled
grep -c '<!--' docs/reference/focus-areas.md docs/reference/paired-reviews.md docs/reference/gaps-identified.md   # 0 each
```

If anything is missing, re-copy the template (Phase 3) and redo Phase 7.

---

## Phase 8: Summary

```
Ralph review prepared:

  Scope:    {one-line scope description}
  Backups:  {"plan #N for YYYY-MM-DD" or "none needed"}

  RALPH.md                           — charter, role protocol, plan critique
  docs/reference/focus-areas.md      — {S} behaviours (2 approvals each)
  docs/reference/paired-reviews.md   — {P} seams (1 approval each), {N} recorded as no seam
  docs/reference/gaps-identified.md  — {G} seeded gaps

  Minimum passes: {4S + 2P}   Submit with: --max-iterations {cap}

Ready to start with /direct-to-ralph --max-iterations {cap}
```
