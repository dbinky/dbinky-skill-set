---
name: ralph-plan
description: Automate ralph review setup — backs up existing tracking docs, copies templates, populates focus areas from branch changes or user-specified scope, and links available design docs into RALPH.md
---

# Ralph Plan

Prepare a project for a ralph refinement run by setting up tracking documents and populating them based on the work scope.

## Usage

```
/ralph-plan                           # Scope = changes in current branch vs main
/ralph-plan PR #123                   # Scope = changes in a specific PR
/ralph-plan "the new auth system"     # Scope = user-described body of work
```

## Process

Follow each phase in order. Do not skip phases.

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

Store the result as `SCOPE` — a description of what's being reviewed plus the list of changed files.

---

## Phase 2: Archive Previous Documents

Check for existing files:
- `RALPH.md`
- `docs/reference/gaps-identified.md`
- `docs/reference/focus-areas.md`

**If none exist:** Skip to Phase 3.

**If any exist:**

1. Determine today's date as `YYYY-MM-DD`.
2. Determine the intra-day plan number — count today's existing backups (using one representative file type so the count is robust regardless of how many files each backup contains) and add 1:
   ```bash
   echo $(( $(ls docs/reference/YYYY-MM-DD-historical-focus-areas-*.md 2>/dev/null | wc -l) + 1 ))
   ```
   If zero exist, the plan number is 1.
3. For each existing file, **move (rename) it** — do NOT copy it. The original must no longer exist at its working-tree location afterward:
   - `RALPH.md` → `docs/reference/YYYY-MM-DD-historical-RALPH-{plan#}.md`
   - `gaps-identified.md` → `docs/reference/YYYY-MM-DD-historical-gaps-identified-{plan#}.md`
   - `focus-areas.md` → `docs/reference/YYYY-MM-DD-historical-focus-areas-{plan#}.md`
4. Report: "Archived existing tracking docs as plan #{plan#} for {date}."

**Why move instead of copy:** Phase 3 lays down fresh templates that carry the required control semantics — notably the `<promise>` tag logic that lets the ralph server stop after repeated "no progress" iterations. Moving the originals out (rather than copying) guarantees they are absent in Phase 3, so a previously mangled `RALPH.md` whose `<promise>` logic was stripped can never survive into the new run.

---

## Phase 3: Copy Templates

After Phase 2, none of the tracking files exist in the working tree — any that existed were moved into the historical archive. Copy the fresh templates from this skill's directory:

- `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates/RALPH.md` → `RALPH.md` (project root)
- `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates/gaps-identified.md` → `docs/reference/gaps-identified.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates/focus-areas.md` → `docs/reference/focus-areas.md`

Create `docs/reference/` if it doesn't exist.

**Always copy `RALPH.md` from the template — never preserve an existing one.** Because Phase 2 archived and removed any prior `RALPH.md`, this guarantees every run starts from a template with intact control semantics (the `<promise>` tag logic), regardless of whether the previous file was legacy-format, customized, or mangled. Anything worth keeping from the old file lives in its archived copy under `docs/reference/`; Phase 6 re-applies the Design Specs links to the fresh file.

---

## Phase 4: Clear Gaps Identified

`docs/reference/gaps-identified.md` was just written from the template in Phase 3, so it is already in the clean state — all sections show `_(none)_` or `_(none yet)_`. Confirm this; no further action is needed.

---

## Phase 5: Populate Focus Areas

Read the instructions at the top of `docs/reference/focus-areas.md` (the "Single-System Review" and "Paired Reviews" sections describe the methodology).

Using the `SCOPE` from Phase 1, analyze the changed files to identify:

### Single-System Review Areas

For each of the following that was touched in the scope, add a row:
- **Domain concepts** — types, models, value objects, business rules
- **Systems** — distinct subsystems or modules (e.g., comfyui adapter, postgres adapter, bot-builder)
- **Apps** — command/entry-point packages (e.g., `cmd/maddie`, `cmd/bot-builder`)
- **APIs** — HTTP handlers, RPC endpoints, WebSocket handlers
- **Contracts** — interfaces, ports, shared types that cross module boundaries

For each, create a table row with:
- A sequential number
- A descriptive name referencing the key files/directories
- Empty Review 1 and Review 2 status columns

### Paired Integration Reviews

Generate all unique pairs from the single-system areas (A+B and B+A are separate pairs since each reviews from a different perspective). For each pair, create a row with:
- A sequential number (continuing from single reviews)
- The pair names and a brief description of the integration seam
- Empty status column

Present the populated tables to the user and ask: "These are the focus areas I identified from the scope. Want to add, remove, or adjust any?"

Apply their feedback, then write the final `docs/reference/focus-areas.md`.

---

## Phase 6: Update RALPH.md Design Specs

Search the project for documentation that's relevant to the scope. Check these directories:
- `docs/specs/`
- `docs/plans/`
- `docs/designs/`
- `docs/superpowers/specs/`
- `docs/superpowers/plans/`

Also check for any `README.md` or design docs referenced in changed files (comments, imports, etc.).

Update the `## Design Specs` section of `RALPH.md` with links to every relevant document found:

```markdown
## Design Specs

- `docs/superpowers/specs/some-design.md` - brief description
- `docs/plans/some-plan.md` - brief description
```

If no documentation is found, note that in the section:

```markdown
## Design Specs

_(no design docs found — consider creating a spec before running the review)_
```

---

## Phase 7: Summary

Show the user what was done:

```
Ralph review prepared:

  Scope: {one-line scope description}
  Backups: {backup status — "plan #N for YYYY-MM-DD" or "none needed"}

  RALPH.md — {created from template / updated design specs}
  docs/reference/focus-areas.md — {N single + M paired = T total reviews}
  docs/reference/gaps-identified.md — cleared for new run

Ready to start with /direct-to-ralph
```
