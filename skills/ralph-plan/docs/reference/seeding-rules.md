# Seeding a Ralph Review Plan

The rules for filling `docs/reference/focus-areas.md` and
`docs/reference/paired-reviews.md`. Shared by `ralph-plan`, `plan-to-ralph` and
`auto-ralph-prep`; the plan critique (`plan-critique.md`) applies the same tests from
outside.

The plan's shape sets the cost of every pass that follows. One branch reviewed as 52
areas and 92 paired rows ran for 235 passes over five days. Under these rules the same
branch is about 15 areas and a dozen seams, and each pass reviews a whole path instead
of a fragment.

## What a review area IS

A single-system area is a **behaviour**: one thing the work promises, traced from the
point where it starts to the point where its effect lands, across every layer in
between. Not a file, not a class, not a layer, not a package.

The wired-and-fed questions that decide every review are path-tracing questions (is it
constructed at the composition root, is it fed real data, does the live path reach the
sink). A reviewer can only answer them by reading the whole path. An area that is one
layer of a path forces the reviewer to read the neighbouring layers anyway, and the
plan then charges again for each neighbour as its own area and again for every seam
between them. One plan reviewed a single feature as seven horizontal slices (handler,
response contract, deck handler, repository, value object, mapping, persistence) and
needed 116 paired rows to stitch them back together. Do not do that.

### How to find the behaviours

1. Read the commit log, the scope, and any spec, design or plan documents. Write down
   what the work **promises**: each capability, guarantee, removal or migration it
   claims to deliver.
2. For each promise, find its **entry point** (a handler, a job, a CLI command, a boot
   step, a config key, a migration) and its **sink** (a table write, a queue publish, a
   response, a rendered artifact, a deleted row, a refused request). Read the code
   between them. That path is the candidate area.
3. Apply the **merge test** to every pair of candidates, then the **split test** to any
   candidate too large for one pass.

### The merge test — two candidates are ONE area when any of these holds

- Judging either one as wired-and-fed requires reading the other. A repository cannot
  be judged "fed" without the handler that feeds it; a gate cannot be judged
  "consulted" without the path that consults it.
- They would be reviewed by reading the same files.
- One of them has no entry point or no sink of its own. A DTO, a mapping, a value
  object, a migration, a constant table, a wiring file: these are hops on some
  behaviour's path, not behaviours.

A change too small to trace on its own — a one-line list edit, a fixture, a config
value, a flag default, a docs page — is **folded in** as a named check of the behaviour
it serves. It does not get an area, because an area costs at least four passes.

### The split test — split only by sub-path, never by layer

If a behaviour is too large for one reviewer to trace in one pass, split it by
**sub-path**: per image kind, per entry point, per tenant type, per lane. Never by
layer. "The render path for canonical images" and "the render path for chat images" are
two areas; "the render handler" and "the render repository" are not.

### Size

The bound is the merge test, not a number. Most work yields 6 to 15 behaviours; a very
large branch may reach 20. Above 20 you are almost certainly slicing by layer or by
file: go back and merge.

## The single-system table and blocks (`focus-areas.md`)

- Five columns exactly: `# | Focus Area | Risk | Review 1 Status | Review 2 Status`.
- **Risk** is `high`, `med` or `low`, ranked on consequence, not size: a silent
  data-corruption, auth-bypass or irreversible-deletion path outranks a cosmetic one
  however much code it touches. Rows are ordered **highest risk first** — the loop takes
  work in this order and may be stopped by its cap, so risk order is what makes a
  truncated run useful.
- Name areas by behaviour, concretely (`a manifest-named asset is on disk and verified
  before the lane serves`). Never `backend`, `misc`, or a file name.
- Under the table, one `### Area N — <name>` block per row with **Promise**, **Entry**,
  **Sink**, **Files**, **Folded in**, **Review 1 — wiring lens** (3–7 numbered checks
  that decide whether the path exists and is fed), **Review 2 — failure lens** (3–7
  numbered checks that decide how it fails, each naming the test that goes red if it is
  wrong) and **Seed evidence** (the greps you ran and what they found, `file:symbol`).
- The two lenses are why an area has two approvals: they ask **different questions**,
  which is what makes the second approval more than a repeat of the first.
- **Numbered lists in the blocks, never checkbox syntax.**
- Delete the templates' `<!-- … -->` example comments in both plan files when you fill them.

## The seam test (`paired-reviews.md`) — a row exists ONLY when all three hold

1. The two behaviours **share** persisted state, a wire contract, a queue or topic, a
   flag, or an **ordering constraint** (X must happen before Y).
2. The two sides were built by **different tasks, phases or commits**. Independent
   authorship is where integration breaks.
3. The failure would be **silent**: each side is green in isolation, and only the
   combination is wrong.

An internal hop inside one behaviour is **not** a seam; it belongs in that area's
checks. Two areas that merely call each other in the same package are not a seam. A
cross-cutting gate (a flag, an auth or consent check) gets a paired row with a governed
behaviour only where the governed side could run **without** the gate and nothing would
go red; otherwise "the path consults the gate at `file:symbol`" is a wiring-lens check
in the governed area.

### One row per seam, both directions, one approval

Each seam is **one row**, named `A ↔ B`, and its single approval covers both
directions: what A assumes about B, and what B assumes about A. **Never emit reverse
rows.** The old format listed every seam twice; 92 of one plan's rows were mirror pairs.

The **Seam** cell is a trace recipe with a stated failure, not a symbol name:
`SHARED: <the state, contract, flag or ordering> · TRACE: <entry, through the seam, to
the observable result> · BREAK: <the failure that must be shown impossible or caught>`.
A row whose BREAK you cannot state is not a seam. Drop it. No `|` inside a cell.

### `## No seam`

List every combination you considered and did not pair, one line each with the reason,
under **Inside one behaviour** (the hop is a check in area N) and **Genuinely
unrelated**. If the budget forces you to drop a real seam, list it under **Deferred
(budget)** with its SHARED and BREAK stated, so a later pass can promote it.

## The plan must fit its budget — a hard constraint

Every box costs **at least two passes**: a refine pass, then a separate verify pass
(the pass that fixes never approves). Singles have two boxes and cannot be batched; a
seam has one. Passes are also spent closing gaps, fixing regressions and re-reviewing,
so real cost runs about 2.6 times the minimum. Require:

> `4 × singles + 2 × paired` **≤ max-iterations × 0.35**

State the arithmetic in RALPH.md's `## Review plan and budget` in exactly this shape
(`auto-ralph-submit` reads the `Iteration cap:` line when no cap is passed to it):

```
- Single-system areas: S (two approvals each)
- Seams: P (one approval each)
- Implied minimum passes: 4 × S + 2 × P = X
- Iteration cap: N
- Budget: N × 0.35 = B — X ≤ B
```

When no cap was given, `N` is the recommended cap `ceil(X / 0.35)`. If a
plan built under the merge test and the seam test still does not fit, defer the
lowest-risk seams first (to `Deferred (budget)`), then merge more; never pad, never pair
layers.

## What every review checks

The recurring failure is code that is present and green in isolation but **never
wired-and-fed** in production. These criteria sit behind every lens check:

- For every new exported field / collaborator / config: `grep -rn <Symbol>` the whole
  repo — is it **constructed or populated outside test files**? If it is only ever set
  in tests, the review fails.
- For every new component: is it assembled into the real composition root so the live
  path reaches it? Name the wiring site, or fail the review.
- Does any production call site pass `nil`/empty/hardcoded where a real producer should
  feed it?
- Do any tests pass only because a fake ignores a load-bearing argument, or because an
  "integration" test asserts an intermediate hop instead of the sink?
- Does the area have coverage across all five scenario categories (happy, success,
  failure, error, edge), and would at least one test **fail** if its production input
  were nil/empty?

Do the greps while seeding and put real findings — `file:symbol` — into each area's
**Seed evidence** and lens checks, rather than generic reminders.

## Seeding `gaps-identified.md`

Seed the `## Open` section with the real gaps you found while seeding — one `- [ ]` item
each, a few lines, `file:symbol` — including any NEW-vs-baseline test failures and any
blockers the implementation reported. Leave the readability rule at the top of the file
intact.
