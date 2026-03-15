---
name: ralph-plan
description: Automate ralph loop setup — backs up existing tracking docs, copies templates, populates focus areas from branch changes or user-specified scope, and links available design docs into RALPH.md
---

# Ralph Plan

Prepare a project for a ralph refinement loop by setting up tracking documents and populating them based on the work scope.

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

## Phase 2: Backup Previous Documents

Check for existing files:
- `docs/reference/gaps-identified.md`
- `docs/reference/focus-areas.md`

**If neither exists:** Skip to Phase 3.

**If either exists:**

1. Determine today's date as `YYYY-MM-DD`.
2. Count existing historical backups for today to determine the intra-day plan number:
   ```bash
   ls docs/reference/YYYY-MM-DD-historical-* 2>/dev/null | wc -l
   ```
   The plan number is `(count / 2) + 1` (since each backup creates 2 files). If zero exist, plan number is 1.
3. For each existing file, copy it:
   - `gaps-identified.md` → `docs/reference/YYYY-MM-DD-historical-gaps-identified-{plan#}.md`
   - `focus-areas.md` → `docs/reference/YYYY-MM-DD-historical-focus-areas-{plan#}.md`
4. Report: "Backed up existing tracking docs as plan #{plan#} for {date}."

---

## Phase 3: Copy Templates

The templates live in this skill's directory. Copy them to the project if they don't already exist:

- `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates/RALPH.md` → `RALPH.md` (project root)
- `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates/gaps-identified.md` → `docs/reference/gaps-identified.md`
- `${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/templates/focus-areas.md` → `docs/reference/focus-areas.md`

Create `docs/reference/` if it doesn't exist.

**If `RALPH.md` already exists** in the project root, do NOT overwrite it — it may have been customized. Instead, just update the Design Specs section in Phase 6.

**If tracking docs already exist** (after backup in Phase 2), overwrite them with fresh templates since we already backed up the originals.

---

## Phase 4: Clear Gaps Identified

Reset `docs/reference/gaps-identified.md` to the clean template state — all sections should show `_(none)_` or `_(none yet)_`. This prepares for a fresh loop run.

If this file was just copied from templates in Phase 3, this is already done. If it existed and was backed up, overwrite it now with the template content.

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

_(no design docs found — consider creating a spec before running the loop)_
```

---

## Phase 7: Summary

Show the user what was done:

```
Ralph loop prepared:

  Scope: {one-line scope description}
  Backups: {backup status — "plan #N for YYYY-MM-DD" or "none needed"}

  RALPH.md — {created from template / updated design specs}
  docs/reference/focus-areas.md — {N single + M paired = T total reviews}
  docs/reference/gaps-identified.md — cleared for new loop

Ready to run the ralph loop with /direct-to-ralph
```
