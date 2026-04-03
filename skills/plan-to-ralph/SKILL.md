---
name: plan-to-ralph
description: Interactive Q&A to generate ralph-o-matic loop files (RALPH.md, focus-areas.md, gaps-identified.md) customized for your project
---

# Plan to Ralph

You are generating ralph-o-matic loop files through interactive Q&A. This skill produces three files that configure a ralph refinement loop:

- `RALPH.md` — Loop instructions (persona, mission, iteration structure, checklist, promise tags)
- `docs/reference/focus-areas.md` — Review tracking table (single areas with 2 passes, paired areas with 1 pass)
- `docs/reference/gaps-identified.md` — Issue tracker (open, fixed, won't-fix sections)

After generating these files, the user runs `/direct-to-ralph` or `ralph-o-matic submit` to start the loop.

## Arguments

Parse the following from the user's command:

- `CONTEXT` (optional): Free-text description of what the loop should focus on (e.g., "on the work on the new identity system"). When provided, narrows codebase scan and pre-seeds Q&A answers.
- `--reset`: Skip backup. Overwrite existing files without creating historical copies.

## Prerequisites

Before starting, verify:

```bash
git rev-parse --is-inside-work-tree
```

If this fails, stop and tell the user: "This skill requires a git repository. Initialize one with `git init` first."

---

## Phase 1: Backup

**Skip this phase if `--reset` was passed.**

Check if any of these files exist:
- `RALPH.md`
- `docs/reference/focus-areas.md`
- `docs/reference/gaps-identified.md`

**If none exist:** Skip silently to Phase 2.

**If any exist:** Back them up before overwriting.

1. Create `docs/reference/historical/` if it doesn't exist.
2. Get today's date as `YYYY-MM-DD`.
3. For each existing file, copy it to `docs/reference/historical/` with the naming pattern:
   - `RALPH.md` → `YYYY-MM-DD-historical-RALPH.md`
   - `focus-areas.md` → `YYYY-MM-DD-historical-focus-areas.md`
   - `gaps-identified.md` → `YYYY-MM-DD-historical-gaps-identified.md`
4. If a same-day backup already exists, append a counter: `-2`, `-3`, etc. For example, if `2026-03-12-historical-RALPH.md` already exists, use `2026-03-12-historical-RALPH-2.md`.
5. Tell the user what was backed up: "Backed up existing files to `docs/reference/historical/`."

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

Ask: "What should this ralph loop review and improve? Describe the feature, system, or area of code."

Store the answer as `MISSION`.

If CONTEXT was provided, use it as the MISSION and tell the user: "Based on your context, the loop will focus on: {CONTEXT}. Sound right?" Accept corrections.

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

The test command is required. Store it as `TEST_COMMAND`.

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

### Question 4: Single Focus Areas

**If the scan found components:**

Generate focus areas across three categories. Think beyond directory structure — identify the domain concepts, their neighborhoods, and the vertical slices that cut through the codebase from top to bottom. Aim for a thorough list; more areas means more coverage.

1. **Components** — Individual modules, packages, or directories that own a distinct responsibility (these come directly from the scan results).
2. **Domain Concepts** — Logical concepts that may span multiple files or directories: a business entity and everything that touches it (models, validation, persistence, API surface, tests), a workflow or process (e.g., "order fulfillment" across handlers, services, and repositories), or a cross-cutting concern (error handling strategy, logging conventions, auth/authz boundaries). Include neighboring concepts — if "payments" is a domain, "refunds" and "invoicing" are its neighbors and deserve their own areas.
3. **Vertical Slices** — End-to-end paths through the system: a user action from API endpoint (or UI event) through business logic, data access, and back. Each slice should trace a complete request/response or workflow from the outermost boundary to the innermost layer and back.

Present all discovered areas as a numbered list grouped by these categories:

```
Based on the codebase scan, here are the candidate focus areas:

**Components:**
  [1] Auth Module (src/auth/) — authentication and session management
  [2] Payment Service (src/payments/) — payment processing logic
  [3] Database Layer (src/db/) — schema, migrations, queries
  [4] Configuration (config/) — env loading, defaults
  [5] CI/CD (.github/workflows/) — build, test, deploy pipelines

**Domain Concepts:**
  [6] User Identity (src/auth/, src/users/, src/db/users.sql) — user model, registration, profile, permissions as a cohesive domain
  [7] Payment Lifecycle (src/payments/, src/refunds/, src/invoicing/) — charge, capture, refund, and invoice as a connected domain neighborhood
  [8] Error Handling Strategy (across all modules) — error types, propagation patterns, user-facing error responses, logging consistency
  [9] Test Quality & Isolation (tests/) — test patterns, mocking strategy, fixture management, coverage gaps

**Vertical Slices:**
  [10] User Registration Flow — POST /register → validation → user creation → welcome email → response
  [11] Payment Checkout Flow — POST /checkout → auth check → inventory hold → charge → confirmation → response
  [12] Admin Dashboard Load — GET /admin → auth middleware → aggregate queries → template rendering → response

Which areas should be included in the review? You can:
- List numbers to include (e.g., "1, 2, 3, 5")
- Say "all" to include everything
- Add areas not in the list (e.g., "also add: Notification System (src/notifications/)")
- Remove areas (e.g., "drop 5")
```

**If the scan found zero components:**

Ask: "I couldn't detect clear components in this codebase. Please describe the focus areas for review — include code modules, domain concepts (business entities and their neighbors), and vertical slices (end-to-end flows). List them one per line, with a brief description of what each covers."

For each selected/added focus area, ensure it has:
- A name
- A category (Component, Domain Concept, or Vertical Slice)
- Key files (from scan or user input)
- A one-line review scope description

Store the final list as `SINGLE_AREAS`.

### Question 5: Paired Focus Areas

Generate suggested pairings from the selected single areas. Pairings should go beyond adjacent modules — think about every meaningful boundary where two areas interact, hand off data, or share assumptions. Generate pairings across all three categories:

1. **Component Integration** — Two components that communicate, share data structures, or depend on each other's contracts. Focus on the seam: data format, error propagation, sequencing, and shared state.
2. **Domain Boundary** — Two domain concepts that neighbor each other or overlap. Focus on whether business rules are consistent across the boundary: does the payment domain's definition of "completed" match what the notification domain expects? Do neighboring concepts handle the same edge cases the same way?
3. **Vertical Slice Handoffs** — A vertical slice paired with a component or domain concept it passes through. Focus on whether the slice's assumptions hold at each layer: does the API contract match the service interface? Does the service call the repository correctly? Does the response shape match what the caller expects?
4. **Cross-Cutting Consistency** — A cross-cutting concern (error handling, logging, config, auth) paired with a specific component or domain to verify the concern is applied consistently and correctly in that area.

When CONTEXT was provided: the context system paired with every component, domain concept, and vertical slice it touches (direct and indirect).

Present as a numbered list grouped by pairing type:

```
Suggested paired reviews (verifying integration and consistency):

**Component Integration:**
  [1] Auth Module + Database Layer — session storage, credential queries, transaction safety
  [2] Payment Service + Auth Module — authorization checks before charges, token propagation

**Domain Boundary:**
  [3] Payment Lifecycle + User Identity — does "active user" mean the same thing in both domains? Refund eligibility vs. account status
  [4] Payment Lifecycle + Error Handling Strategy — are payment failures surfaced consistently? Retry semantics, idempotency

**Vertical Slice Handoffs:**
  [5] User Registration Flow + Database Layer — user creation query correctness, constraint handling, rollback on email failure
  [6] Payment Checkout Flow + Payment Lifecycle — does the flow exercise the full lifecycle? Edge cases at each layer boundary

**Cross-Cutting Consistency:**
  [7] Error Handling Strategy + Payment Service — are payment errors typed, logged, and surfaced correctly?
  [8] Configuration + Auth Module — do auth config values (token TTL, allowed origins) match runtime behavior?

Which pairs to include? Same controls as above: list numbers, "all", add new pairs, or drop.
```

Store the final list as `PAIRED_AREAS`.

### Question 6: Checklist

Present the proposed checklist:

```
Proposed review checklist (assessed each iteration):

Universal (always included):
  [1] All tests pass (`{TEST_COMMAND}`)
  [2] No open issues in `docs/reference/gaps-identified.md` for this focus area
  [3] The focus area is complete and polished — you'd be proud to ship it

Proposed (based on your project):
  [4] {proposed item based on context, e.g., "The code aligns with the design doc"}
  [5] {proposed item, e.g., "Module boundaries are clean — no adapter types in core"}

Add, remove, or edit items? Or accept as-is?
```

Generate 1-3 proposed items based on what you've learned:
- If a design doc exists → add spec alignment item
- If hexagonal/layered architecture detected → add boundary cleanliness item
- If security-related MISSION → add security-specific item
- If test framework detected with coverage tools → add test scenario categories item

Store the final checklist as `CHECKLIST`.

### Question 7: Additional Constraints

Present the standard constraints that are always included:

```
These constraints are always included in the loop:
- No sub-agents for bulk generation
- Read before you write
- One focus area per iteration
- Don't invent new functionality (log gaps instead)

Any project-specific constraints to add? (e.g., "Do NOT write Python scripts for analysis", "Run linting after every change"). Enter to skip.
```

Store any additions as `EXTRA_CONSTRAINTS`.

---

## Phase 4: Generate

Write all 3 files. Create `docs/reference/` directory if it doesn't exist.

**Before generating, read the example files for exact formatting reference:**
- `${CLAUDE_PLUGIN_ROOT}/skills/plan-to-ralph/docs/reference/example-focus-areas.md` — canonical format for focus-areas.md (table layout, review guidance sections, HTML structure)
- `${CLAUDE_PLUGIN_ROOT}/skills/plan-to-ralph/docs/reference/example-gaps-identified.md` — canonical format for gaps-identified.md (section structure, HTML comment examples showing how entries should be formatted)

Match the formatting, section structure, and HTML comment patterns from these examples exactly. The inline templates below define the content structure; the example files define the precise formatting.

### Generate RALPH.md

Write `RALPH.md` in the repository root with this structure (substitute all `{VARIABLES}` with values from Q&A):

```markdown
# Loop Instructions

You are in an automated prompt loop and the user is unavailable for input, so you should just do the work without asking for any user input.

## Persona

{PERSONA}

## Your Mission

{MISSION_DESCRIPTION — expand the MISSION into 1-3 paragraphs describing what was built, what the loop is reviewing, and what "done" looks like}

## Tracking System

- At the start of an iteration, read `docs/reference/focus-areas.md`. Each single area (from table 1) needs **2 review passes** before being considered __done__. Each paired area (from table 2) needs **1 review pass** before being considered __done__. Use this document to track which reviews have been advanced and completed.
- DO NOT update this document until the "Wrap Up" phase at the end of each iteration. Updating is conditional on your checklist assessment.

## Constraints

- **Do NOT use sub-agents for bulk generation.** When you modify or create code, do it by hand, one component at a time, with thought behind each decision.
- **Read before you write.** Before modifying any file, read the relevant sections. Before claiming something is fine, read it and reason about quality.
- **One focus area per iteration.** Don't try to fix everything at once. Pick the most important remaining issue within the focus area, fix it well, then let the next iteration handle the next issue.
- **Do NOT invent new functionality to fill perceived gaps.** Maintain a list of things you find that should be fixed at `docs/reference/gaps-identified.md` in the `## Open Issues` section. If you perceive there is new, missing functionality beyond the current scope, log it in the `## Won't Fix (Beyond Current Scope)` section. If something on the list has been fixed in a previous loop, move it to `## Fixed Previously`.
{EXTRA_CONSTRAINTS — each as a new bullet point with bold lead, same format as above}

## Iteration Structure

1. **Read the Tracking File** — Read `docs/reference/focus-areas.md` and pick a review focus area that hasn't been completed yet. Complete single area reviews before moving to paired area reviews.
2. **Read the Area's Code** — Deeply examine the code for the chosen focus area. Read every file. Understand the patterns.
3. **Analyze findings and update the gaps list** — Cross-reference what you just read with the design doc and codebase conventions. Add any issues found to `docs/reference/gaps-identified.md` in the `## Open Issues` section.
4. **Fix the single most important issue** — Fix it well. If the issue spans multiple files, fix all of them consistently. Once fixed, move the issue to `## Fixed Previously` in `docs/reference/gaps-identified.md`.
5. **Run the tests** — Run `{TEST_COMMAND}`. Investigate and fix each failure.
6. **Assess the checklist** — Evaluate honestly, then proceed to the Wrap Up phase.

## The Checklist (be brutally honest)

Do NOT check a box unless you could defend it in a code review:

{CHECKLIST — each item as `- [ ] {item text}`}

## Wrap Up

This is the final phase of every iteration. Follow these steps in order.

**Step A — Determine if this focus area passed review.**

All checklist boxes must be honestly checked for the focus area to pass. There is no partial credit. Don't feel bad if it didn't pass — that just means the next iteration will take another crack at it. Improving things each cycle is a successful outcome.

**Step B — Update tracking (only if the focus area passed).**

- If the focus area **passed**: Mark that focus area's review as complete in `docs/reference/focus-areas.md`.
- If the focus area **did not pass**: Do NOT update `docs/reference/focus-areas.md`.

**Step C — Commit and push your branch to remote.**

Always commit and push, whether the focus area passed or not. Your work this iteration is valuable either way.

**Step D — Output your promise tag and stop.**

Check `docs/reference/focus-areas.md`. Are ALL reviews (single areas and paired areas) now marked complete?

- If **all reviews are complete**: output `<promise>FINIT</promise>`
- If **any reviews remain incomplete**: output `<promise>CLOSER</promise>`

Output exactly one `<promise>` tag, then stop. Do not output anything after the tag. The next iteration will carry on your good work.
```

### Generate docs/reference/focus-areas.md

Write `docs/reference/focus-areas.md` with this structure:

```markdown
# Focus Areas Review Tracking

This document tracks the systematic review of {MISSION_SHORT — one-line summary}.

Each **single focus area** requires **2 full review passes** before being considered done.
Each **paired focus area** requires **1 full review pass** before being considered done.

A review pass is only checked when the checklist produces a passing result for that area.

---

## Table 1: Single Area Reviews (2 passes each)

| # | Focus Area | Pass 1 | Pass 2 | Status |
|---|---|---|---|---|
{For each SINGLE_AREA, numbered starting at 1:}
| {N} | {Name} (`{key files}` — {review scope description}) | [ ] | [ ] | |

---

## Table 2: Paired Area Reviews (1 pass each)

| # | Pair | Pass | Status |
|---|---|---|---|
{For each PAIRED_AREA, numbered continuing from single areas:}
| {N} | {Area 1} + {Area 2} ({seam description}) | [ ] | |

---

## Review Guidance

### Single Area Reviews — What To Check

**Pass 1 (Correctness & Coverage):**
- All public functions have test coverage (happy path, failure, error, edge cases)
- The code does what the design doc says it should
- Error handling is thorough — no silent failures
- Module boundaries are clean (no leaking types across boundaries)

**Pass 2 (Robustness & Extensibility):**
- Code aligns to the spirit of the design, not just surface conformance
- The component is ready to be extended by future work
- Test assertions test real risks, not just happy-path ceremony
- Edge cases are handled (empty inputs, concurrent access, boundary values)

### Paired Area Reviews — What To Check

Paired reviews verify that two components integrate correctly:
- Data flows cleanly across the boundary
- Error propagation works end-to-end
- No assumptions in one component that the other doesn't satisfy
- The integration tests cover the seams between the pair
```

### Generate docs/reference/gaps-identified.md

Write `docs/reference/gaps-identified.md`:

```markdown
# Gaps Identified

Issues found during review iterations. Only the user can move items to the Won't Fix sections.

## Open Issues

_(none)_

## Fixed Previously

_(none yet)_

## Won't Fix (Beyond Current Scope)
```

---

## Phase 5: Review

Show the user a summary:

```
Ralph loop files generated:

  RALPH.md
    Persona:      {first sentence of PERSONA}
    Test command:  {TEST_COMMAND}
    Checklist:     {N} items

  docs/reference/focus-areas.md
    Single areas:  {N} (2 passes each)
    Paired areas:  {N} (1 pass each)
    Total reviews: {TOTAL} passes

  docs/reference/gaps-identified.md
    Empty template ready

Want to review or edit any of the generated files before I commit?
```

If they want to review a file, show its content using the Read tool. If they make manual edits (outside Claude Code), re-read the file to acknowledge changes. If no review needed, proceed to Phase 6.

---

## Phase 6: Commit

Stage and commit all generated files plus any historical backups:

```bash
git add RALPH.md docs/reference/focus-areas.md docs/reference/gaps-identified.md
# Also stage historical backups if they were created
git add docs/reference/historical/ 2>/dev/null || true
git commit -m "chore: generate ralph loop files via plan-to-ralph"
```

Tell the user:

```
Loop files committed. To start the ralph loop:

  /direct-to-ralph "{MISSION_SHORT}"

Or submit manually:

  ralph-o-matic submit
```
