# Gaps Identified

**Keep this file readable — every pass reads its open list.** Records are at most
ten lines with `file:symbol`. A closed item moves under `## Closed` with a one-line
note; its history does not stay in the open list. (One ledger grew to 1.4 MB and
18,000 lines because every pass appended a multi-page record, and no reviewer could
fit it in context.)

Format for an open item:

- [ ] G<n> **ISSUE TITLE** — what is wrong, where (`file:symbol`), and enough context for the next pass to act without rediscovering it.

Ids (`G1`, `G2`, …) are never reused. Only a verify pass closes a gap; a refiner that fixed
one adds `- G<n> · Recommend: close · <sha> · <what fixed it>` under `## Pending approvals`.

Tags:
- `[NOT-WIRED]` — a production path that is present but unfed: constructed only in tests, or fed `nil`/empty/hardcoded. An inert path is an open gap, never "acceptable".
- `[GREEN-THEATER]` — a test that passes only because a fake ignores a load-bearing argument, or an "integration" test that asserts an intermediate hop instead of the sink.
- `[REGRESSION]` — a test that was not failing at the branch-point baseline and now fails, or something that compiled at baseline and no longer does. Fixed before anything else.

## Open

_(none)_

## Pending approvals

_(none)_

## Escalations

Questions only a human can answer, set aside so they stop blocking passes. Plain
lines, no checkboxes — these do not block `<promise>FINIT</promise>`, but every
pass that ends the loop lists them.

_(none)_

## Closed

_(none yet)_

## Won't Fix

Out of scope for this work, or pre-existing baseline failures this work touched.

_(none)_
