# Critiquing a Seeded Ralph Review Plan

Run this immediately after seeding, **before** the loop starts. A plan reviewed only by
its author keeps its author's slicing, so where the calling skill can, run it as a fresh
subagent that did not write the plan. Work from the code, not from the plan's prose.

Inputs: `RALPH.md`, `docs/reference/focus-areas.md`, `docs/reference/paired-reviews.md`,
`docs/reference/gaps-identified.md`, the spec/design/plan docs the plan cites, the
seeder's report of which candidates it merged and why, and the iteration cap.

The rules are in `seeding-rules.md` (same directory). Apply them from outside:

## 1. The merge test (areas)

Two areas are ONE area when judging either as wired-and-fed requires reading the other,
when they would be reviewed by reading the same files, or when one has no entry point or
no sink of its own. **Look especially for layer slicing**: several areas whose names are
a handler, a contract, a repository, a value object and a mapping of the same feature.
Merge them into the behaviour they implement, named by what it does. Fold changes too
small to trace into the behaviour they serve as **Folded in** checks. Split only by
sub-path, never by layer.

## 2. The block test

Every area needs a `### Area N — <name>` block with Promise, Entry, Sink, Files, Folded
in, Review 1 — wiring lens, Review 2 — failure lens, and Seed evidence. Fill any missing
block or empty field from the code. A lens with fewer than three concrete checks, or
checks that are generic reminders rather than `file:symbol` claims, is not finished:
finish it.

## 3. The seam test (rows)

Keep a row only when the two behaviours share state, a contract, a queue, a flag or an
ordering; were built by different tasks, phases or commits; and would break silently.
Strike internal hops (move the check into that area's lens), rows between areas that
merely call each other, rows with no stated or observable BREAK, and every **reverse
row** — merge `A as impacted by B` and `B as impacted by A` into one `A ↔ B` row whose
Seam cell states what each side assumes about the other. Every struck seam goes under
`## No seam` with a reason; every missing real seam gets added with SHARED, TRACE and
BREAK.

## 4. Size and budget

Most work yields 6 to 15 areas; above 20 is layer slicing until proven otherwise.
Recompute `4 × singles + 2 × paired` and rewrite the arithmetic in RALPH.md. If it still
exceeds `max-iterations × 0.35`, move the lowest-risk seams to `Deferred (budget)`.

## 5. Risk order

Both tables in descending risk. Reorder only where a rank is indefensible on consequence,
and say why.

## Never

- Tick or untick a box. Touch production code or tests.
- Add reverse rows, checkbox syntax outside the two tables, a sixth column, or a `|` in a cell.
- Remove or reword RALPH.md's role sections, Wrap Up, promise tags or definition of done.
- Merge two behaviours that fail the merge test just to shrink the list. The bound is the test, not a number.

Renumber both tables after merges and strikes, and update every `### Area N` heading,
every `A ↔ B` name and every cross-reference to match.

## Record what you did

Append a `## Plan critique` section to `RALPH.md`:

- `areas A → B, seams C → D, implied passes E → F, budget G`
- one line per merge (which areas became which, which merge-test clause applied)
- one line per struck seam and its reason; one line per added seam
- one line per area block you completed
- `No changes needed` for any test that found nothing, so a reader can tell a clean plan
  from an unchecked one
