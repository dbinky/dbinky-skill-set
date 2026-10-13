# Focus Areas — Single-System Reviews

Each area is a **behaviour**: one thing this work promises, traced from where it
starts (Entry) to where its effect lands (Sink), across every layer in between —
not a file, a class, a layer or a package. How areas are chosen:
`${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/seeding-rules.md`.

Each area needs **two approvals**: Review 1 (the **wiring** lens — does the path
exist and is it fed?) and Review 2 (the **failure** lens — how does it fail?). Only
a verify pass ticks a box; see `RALPH.md`.

Rows are ordered **highest risk first**. Keep the table's five columns exactly. The
detail blocks use numbered lists, never checkbox syntax — `[ ]`/`[x]` outside the
table corrupts the approval count.

## Single-System Reviews

| # | Focus Area | Risk | Review 1 Status | Review 2 Status |
|---|---|---|---|---|

<!-- One row per behaviour, named by what it does, e.g.:
| 1 | an existing user with incomplete preferences is gated to setup and back | high | [ ] | [ ] |
-->

## Area details

<!-- One block per row:

### Area 1 — <behaviour name>

- **Promise:** <one sentence: what the work says this does>
- **Entry:** `file:symbol`
- **Sink:** `file:symbol`
- **Files:** <every file on the path — changed files first, then unchanged files the path crosses>
- **Folded in:** <each small change reviewed as part of this behaviour: file and what to check>
- **Review 1 — wiring lens:**
  1. <constructed at the composition root at `file:symbol`>
  2. <fed by the real producer at `file:symbol`, not nil/empty/hardcoded>
  3. <the happy and success paths reach the sink; the test that shows it>
- **Review 2 — failure lens:**
  1. <nil/empty input → expected outcome; the test that goes red if wrong>
  2. <timeout / partial failure / rollback>
  3. <concurrency or ordering that must hold>
- **Seed evidence:** <greps run while seeding and what they found, `file:symbol`>
-->

## Pending approvals

<!-- Refiners add one line here per pass; verifiers remove it. Plain lines, no checkboxes:
- A1/R1 · Recommend: approve · <sha> · <evidence, file:symbol>
-->
