# Paired Reviews — Integration Seams

A row exists only for a real **seam**: the two behaviours share persisted state, a
wire contract, a queue, a flag or an ordering constraint; they were built by
different tasks, phases or commits; and a break would be silent (each side green in
isolation). How seams are chosen:
`${CLAUDE_PLUGIN_ROOT}/skills/ralph-plan/docs/reference/seeding-rules.md`.

Each seam is **one row**, named `A ↔ B`, and its single approval covers **both
directions**. There are no reverse rows. Paired reviews start only once every
single-system box in `focus-areas.md` is ticked.

Rows are ordered **highest risk first**. Keep the five columns exactly; no `|`
inside a cell.

## Paired Integration Reviews

| # | Paired Review | Seam | Risk | Status |
|---|---|---|---|---|

<!-- e.g.:
| 1 | <Area A name> ↔ <Area B name> | SHARED: <the state, contract, flag or ordering> · TRACE: <entry, through the seam, to the observable result> · BREAK: <the failure that must be shown impossible or caught> | high | [ ] |
-->

## No seam

Every combination considered and not paired, one line each with the reason. An
omission written down is a decision; a silent one is a coverage gap nobody can audit.

### Inside one behaviour

<!-- - <A> / <B> — the hop is check N of Area <N>'s wiring lens -->

### Genuinely unrelated

<!-- - <A> / <B> — no shared state, contract, flag or ordering -->

### Deferred (budget)

<!-- - DEFERRED (budget): <A> ↔ <B> — SHARED: … · BREAK: … -->

## Pending approvals

<!-- Refiners add one line here per pass; verifiers remove it. Plain lines, no checkboxes:
- S1 · Recommend: approve · <sha> · <evidence, file:symbol>
-->
